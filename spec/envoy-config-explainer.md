# Envoy Configuration Explainer

This document walks through the Envoy config produced by `charts/mtls-forward-proxy/templates/envoy-configmap.yaml`, explaining what each block does and why it is configured the way it is.

---

## Overview

The config is divided into four top-level sections:

```
bootstrap_extensions   — one-time feature flags enabled at startup
admin                  — local management interface
static_resources
  listeners            — inbound connection handlers
  clusters             — outbound connection targets
```

All configuration is static (no xDS management server). Envoy reads this file once at startup and does not reload dynamically. A change to any value that affects the ConfigMap triggers a rolling restart via a `checksum/config` annotation on the Deployment.

---

## 1. Bootstrap Extensions

```yaml
bootstrap_extensions:
  - name: envoy.bootstrap.internal_listener
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.bootstrap.internal_listener.v3.InternalListener
```

Enables Envoy's **internal listener** feature. Without it, listeners can only bind to real network sockets. With it, Envoy can create virtual listeners that live entirely in memory — traffic can be routed between them without leaving the process.

This is required for the TLS interception architecture used here: the CONNECT tunnel from the browser is handed off to an internal listener that terminates the browser's TLS, all within the same Envoy instance.

---

## 2. Admin Interface

```yaml
admin:
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 9901
```

The admin interface provides a local HTTP API for inspecting Envoy's state — active connections, cluster health, configuration dumps, and metrics. Bound to `127.0.0.1` only so it is never reachable from outside the pod.

Useful endpoints when debugging:
- `GET /ready` — is Envoy ready to serve traffic?
- `GET /clusters` — cluster health and endpoint state
- `GET /stats` — all counters and gauges, including `dns_cache.*`
- `GET /config_dump` — full running configuration

---

## 3. Listeners

This config has two listeners: one bound to a real network port, and one that is internal-only.

---

### Listener 1: `forward_proxy`

```yaml
- name: forward_proxy
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 3128
```

The **entry point**. Binds on all interfaces on port 3128 (the conventional explicit proxy port). Clients (browsers, curl) are configured to send all their traffic here. No `transport_socket` — it accepts **plain TCP**. With an explicit proxy the client must send a plaintext `CONNECT` request before any TLS can happen.

#### upgrade_configs: CONNECT

```yaml
upgrade_configs:
  - upgrade_type: CONNECT
```

Tells the HCM to accept the `CONNECT` method. By default Envoy rejects CONNECT — this enables forward proxy tunnel behaviour. When a browser sends `CONNECT api.example.com:443 HTTP/1.1`, Envoy handles it rather than returning 405.

#### Access Logging

```yaml
access_log:
  - name: envoy.access_loggers.file
    typed_config:
      path: /dev/stdout
      log_format:
        text_format: "[%START_TIME%] \"%REQ(:METHOD)% ..."
```

Every request logged to stdout, visible in `kubectl logs`. Records timestamp, method, path, protocol, response code, bytes sent, host, and user-agent.

#### Route Config

```yaml
virtual_hosts:
  - name: all_hosts
    domains: ["*"]
    routes:
      - match:
          connect_matcher: {}
        route:
          cluster: internal_tls_wildcard
          upgrade_configs:
            - upgrade_type: CONNECT
              connect_config: {}
```

A single catch-all virtual host matches every hostname. The `connect_matcher` matches only `CONNECT` method requests — any CONNECT tunnel (regardless of hostname) is routed to the `internal_tls_wildcard` cluster, which delivers it to the internal TLS-terminating listener.

`connect_config: {}` tells Envoy to **terminate** the CONNECT — it responds `200 OK` and the tunnel is established. Bytes from the browser after that point are forwarded into the internal listener.

#### Dynamic Forward Proxy filter

```yaml
http_filters:
  - name: envoy.filters.http.dynamic_forward_proxy
    typed_config:
      "@type": ...FilterConfig
      dns_cache_config:
        name: dynamic_dns_cache
        dns_lookup_family: V4_ONLY
```

The DFP filter is present on this listener to populate the shared DNS cache as connections arrive. The actual DNS-based routing happens in Listener 2 and the DFP cluster.

---

### Listener 2: `tls_termination_wildcard`

```yaml
- name: tls_termination_wildcard
  internal_listener: {}
```

An **internal listener** — no network socket. Traffic reaches it only from other parts of Envoy via the `internal_tls_wildcard` cluster. Its job is to terminate the browser's TLS using the wildcard cert, then route the decrypted HTTP to the dynamic forward proxy cluster.

#### Downstream TLS context

```yaml
transport_socket:
  name: envoy.transport_sockets.tls
  typed_config:
    "@type": ...DownstreamTlsContext
    common_tls_context:
      tls_certificates:
        - certificate_chain:
            filename: /etc/envoy/certs/tls.crt
          private_key:
            filename: /etc/envoy/certs/tls.key
      tls_params:
        tls_minimum_protocol_version: TLSv1_2
        tls_maximum_protocol_version: TLSv1_3
```

Envoy acts as TLS **server** toward the browser, presenting the `*.example.com` wildcard cert (`tls.crt` / `tls.key` — cert-manager's standard key names). Because it is a wildcard, this single cert covers every `*.example.com` hostname — no per-hostname certs are needed. The cert is mounted from the `envoy-downstream-certs` Secret, issued by cert-manager's `lab-ca-issuer`.

TLS 1.0 and 1.1 are not permitted.

#### Route Config — hostOverrides

```yaml
virtual_hosts:
  {{- range .Values.envoy.hostOverrides }}
  - name: {{ .hostname | replace "." "-" }}
    domains: ["{{ .hostname }}", "{{ .hostname }}:443"]
    routes:
      - match:
          prefix: "/"
        route:
          cluster: dynamic_forward_proxy_cluster
        typed_per_filter_config:
          envoy.filters.http.dynamic_forward_proxy:
            "@type": ...PerRouteConfig
            host_rewrite_literal: {{ .address }}
  {{- end }}
  - name: all_backends
    domains: ["*"]
    routes:
      - match:
          prefix: "/"
        route:
          cluster: dynamic_forward_proxy_cluster
```

For hostnames listed in `hostOverrides` (e.g. `api.example.com`), the route carries a `typed_per_filter_config` that rewrites the upstream hostname to the K8s service address (e.g. `mtls-webserver.default.svc.cluster.local`) **before** the DFP filter performs its DNS lookup. This is necessary because client-facing hostnames like `api.example.com` have no in-cluster DNS entry — without the rewrite, DFP would fail to resolve them.

The rewrite must be in `typed_per_filter_config`, not on the route action's `host_rewrite_literal`. The DFP filter runs before the router, so only per-filter config is visible to it at DNS lookup time.

For any hostname not in `hostOverrides`, the catch-all `all_backends` virtual host forwards to the DFP cluster without rewriting — DNS resolution uses the original hostname directly. This handles any public external service without any config changes.

The backend server cert must include both the public hostname **and** the K8s service FQDN as SANs, because Envoy validates the upstream cert against the rewritten (service FQDN) hostname.

#### Dynamic Forward Proxy filter (Listener 2)

```yaml
http_filters:
  - name: envoy.filters.http.dynamic_forward_proxy
    typed_config:
      dns_cache_config:
        name: dynamic_dns_cache
        dns_lookup_family: V4_ONLY
```

Shares the same named DNS cache (`dynamic_dns_cache`) as Listener 1. Performs the actual DNS resolution of the upstream hostname (after any `host_rewrite_literal` override) and sets the endpoint for the router to forward to.

---

## 4. Clusters

---

### `internal_tls_wildcard`

```yaml
- name: internal_tls_wildcard
  load_assignment:
    endpoints:
      - lb_endpoints:
          - endpoint:
              address:
                envoy_internal_address:
                  server_listener_name: tls_termination_wildcard
```

No network address — uses `envoy_internal_address` to route traffic to the `tls_termination_wildcard` internal listener by name. This is what connects the two listeners together: Listener 1 routes CONNECT tunnels here, and this cluster delivers them to Listener 2. No transport socket because the connection never leaves the Envoy process.

---

### `dynamic_forward_proxy_cluster`

```yaml
- name: dynamic_forward_proxy_cluster
  lb_policy: CLUSTER_PROVIDED
  cluster_type:
    name: envoy.clusters.dynamic_forward_proxy
    typed_config:
      "@type": ...ClusterConfig
      dns_cache_config:
        name: dynamic_dns_cache
        dns_lookup_family: V4_ONLY
  transport_socket:
    name: envoy.transport_sockets.tls
    typed_config:
      "@type": ...UpstreamTlsContext
      common_tls_context:
        tls_certificates:
          - certificate_chain:
              filename: /etc/envoy/client-certs/tls.crt
            private_key:
              filename: /etc/envoy/client-certs/tls.key
        validation_context:
          trusted_ca:
            filename: /etc/envoy/upstream-ca/tls.crt
        tls_params:
          tls_minimum_protocol_version: TLSv1_2
          tls_maximum_protocol_version: TLSv1_3
```

The single upstream cluster for all backends. `lb_policy: CLUSTER_PROVIDED` and the `dynamic_forward_proxy` cluster type work together — the DFP filter resolves the hostname via DNS and tells the cluster which endpoint to use at request time. No static endpoint list is needed.

**`tls_certificates`** — This is the **mTLS** part. Envoy presents `tls.crt` (CN=scanner_sysid, issued by lab-ca) to every backend during the TLS handshake. The backend verifies it and sees the client identity. The cert is mounted from the `envoy-client-certs` Secret, issued by cert-manager's `lab-ca-issuer`.

**`validation_context.trusted_ca`** — Envoy verifies each backend's server cert against `tls.crt` from the `envoy-upstream-ca` Secret. This Secret contains only the public cert of the webserver CA (extracted from cert-manager's `cert-manager-webserver-ca-new` Secret — no private key). All backend server certs are issued by `webserver-ca-issuer`, so they all pass this check.

Adding a new backend requires no changes here — as long as its server cert is issued by `webserver-ca-issuer`, Envoy will trust it automatically.

---

## Data Flow

```
Browser
  │
  │  CONNECT api.example.com:443 HTTP/1.1   (plaintext → port 3128)
  ▼
[Listener 1: forward_proxy]
  │  catch-all virtual host, connect_matcher → internal_tls_wildcard cluster
  │  DFP filter populates dns_cache
  │  200 OK (tunnel established)
  │
  │  <TLS ClientHello from browser>
  ▼
[Cluster: internal_tls_wildcard]
  │  envoy_internal_address → tls_termination_wildcard listener
  ▼
[Listener 2: tls_termination_wildcard]  (internal, no network socket)
  │  DownstreamTlsContext: presents *.example.com wildcard cert to browser
  │  TLS handshake completes; browser traffic is now decrypted
  │
  │  GET / HTTP/1.1 (plaintext HTTP, now visible to Envoy)
  │  hostOverride match: api.example.com → host_rewrite_literal: mtls-webserver.default.svc.cluster.local
  │  DFP filter resolves mtls-webserver.default.svc.cluster.local via DNS
  │  route → dynamic_forward_proxy_cluster
  ▼
[Cluster: dynamic_forward_proxy_cluster]
  │  endpoint: resolved IP of mtls-webserver service, port 443
  │  UpstreamTlsContext:
  │    - presents tls.crt (CN=scanner_sysid) to backend   ← mTLS client cert
  │    - verifies backend cert against webserver-ca tls.crt
  ▼
nginx backend
  │  ssl_verify_client on → verifies client cert against lab-ca ✓
  │  serves response
  ▼
Response travels back through the same chain to the browser
```

For a backend with no `hostOverride` (e.g. a public SaaS API), the flow is identical except the DFP filter resolves the original hostname directly via external DNS — no rewrite step.

---

## Key Concepts

| Concept | Where used | What it means |
|---------|-----------|---------------|
| `DownstreamTlsContext` | Listener 2 | Envoy acts as TLS **server** toward the browser |
| `UpstreamTlsContext` | DFP cluster | Envoy acts as TLS **client** toward every backend |
| `tls_certificates` in upstream | DFP cluster | Envoy presents a client certificate — this is the **mTLS** part |
| `validation_context` | DFP cluster | Envoy verifies every backend's server cert against the webserver CA |
| Wildcard cert | Listener 2 | One `*.example.com` cert covers all backends — no per-hostname certs |
| `connect_matcher` | Listener 1 | Matches HTTP `CONNECT` method only |
| `connect_config: {}` | Listener 1 | Terminates the CONNECT — Envoy responds 200 and owns the tunnel |
| `envoy_internal_address` | `internal_tls_wildcard` | Routes to an internal listener, not a network socket |
| `internal_listener: {}` | Listener 2 | Virtual listener with no network binding |
| Dynamic forward proxy | Listener 2 + DFP cluster | Resolves upstream hostname via DNS at request time — no static upstream list |
| `host_rewrite_literal` | `typed_per_filter_config` | Rewrites hostname before DFP DNS lookup — required for in-cluster services |
| `hostOverrides` | values.yaml | Maps client-facing hostname to K8s service FQDN for in-cluster backends |
