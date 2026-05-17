# Envoy Configuration Explainer

This document walks through `envoy-proxy/envoy.yaml` section by section, explaining what each block does and why it is configured the way it is.

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

All configuration is static (no xDS management server). Envoy reads this file once at startup and does not reload dynamically.

---

## 1. Bootstrap Extensions (lines 1–4)

```yaml
bootstrap_extensions:
  - name: envoy.bootstrap.internal_listener
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.bootstrap.internal_listener.v3.InternalListener
```

This enables Envoy's **internal listener** feature. Without it, listeners can only bind to real network sockets. With it, Envoy can create virtual listeners that live entirely in memory — traffic can be routed between them without leaving the process.

This is required for the TLS interception architecture used here: the CONNECT tunnel from the browser is handed off to an internal listener that terminates the browser's TLS, all within the same Envoy instance.

---

## 2. Admin Interface (lines 6–10)

```yaml
admin:
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 9901
```

The admin interface provides a local HTTP API for inspecting Envoy's state — active connections, cluster health, configuration dumps, and metrics. It is bound to `127.0.0.1` only, so it is never reachable from outside the pod. Binding it to `0.0.0.0` would expose configuration and traffic control endpoints to the network, which would be a significant security risk.

Useful endpoints when debugging:
- `GET /ready` — is Envoy ready to serve traffic?
- `GET /clusters` — what clusters exist and are their endpoints healthy?
- `GET /config_dump` — what is the full running configuration?

---

## 3. Listeners

Listeners are where Envoy accepts incoming connections. This config has two: one bound to a real network port, and one that is internal-only.

---

### Listener 1: `forward_proxy` (lines 19–82)

```yaml
- name: forward_proxy
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 3128
```

This is the **entry point**. It binds on all interfaces on port 3128, which is the conventional port for an explicit HTTP proxy. Clients (browsers, curl) are configured to send all their traffic here.

The listener has no `transport_socket`, meaning it accepts **plain TCP** — no TLS at this level. Clients connect in cleartext, which is intentional. With an explicit proxy, the client first needs to send a plaintext `CONNECT` request or a plain HTTP `GET` before any TLS can happen.

#### HTTP Connection Manager filter (lines 26–82)

```yaml
- name: envoy.filters.network.http_connection_manager
```

The HCM is Envoy's main HTTP processing filter. It parses the incoming TCP stream as HTTP, applies routing logic, and forwards requests to clusters. Everything below it describes how that HTTP parsing and routing works.

```yaml
codec_type: HTTP1
```

Forces HTTP/1.1 parsing. Browsers use HTTP/1.1 when talking to an explicit proxy — they don't use HTTP/2 for the CONNECT request itself.

```yaml
upgrade_configs:
  - upgrade_type: CONNECT
```

This tells the HCM to accept the `CONNECT` method. By default Envoy rejects CONNECT — this line enables the forward proxy tunnel behaviour. When a browser sends `CONNECT api.example.com:443 HTTP/1.1`, Envoy handles it rather than returning 405.

#### Access Logging (lines 33–39)

```yaml
access_log:
  - name: envoy.access_loggers.file
    typed_config:
      path: /dev/stdout
      log_format:
        text_format: "[%START_TIME%] \"%REQ(:METHOD)% ..."
```

Every request is logged to stdout in a format similar to nginx/Apache combined log. Writing to `/dev/stdout` means the logs appear in `kubectl logs` output. The format records the timestamp, method, path, protocol, response code, bytes sent, host header, and user-agent.

#### Route Config (lines 40–78)

The route config is the forwarding table — it maps incoming requests to upstream clusters based on the `Host` header (for plain HTTP) or the CONNECT authority (for tunnels).

```yaml
virtual_hosts:
  - name: api_example_com
    domains: ["api.example.com", "api.example.com:80", "api.example.com:443"]
```

A virtual host matches on the `Host` header value. All three domain entries are needed because:
- `api.example.com` — matches plain HTTP GET requests (`Host: api.example.com`)
- `api.example.com:80` — matches when curl uses `--proxytunnel` with an `http://` URL (sends `CONNECT api.example.com:80`)
- `api.example.com:443` — matches browser HTTPS (sends `CONNECT api.example.com:443`)

The `app.example.com` and `auth.example.com` virtual hosts only need `:443` because those are external hosts that browsers would only reach over HTTPS.

#### Two routes for `api.example.com`

```yaml
routes:
  # CONNECT tunnel → internal TLS-terminating listener
  - match:
      connect_matcher: {}
    route:
      cluster: internal_tls_api
      upgrade_configs:
        - upgrade_type: CONNECT
          connect_config: {}

  # Plain HTTP GET → upstream mTLS directly
  - match:
      prefix: "/"
    route:
      cluster: cluster_api_example_com
```

Routes are evaluated top-to-bottom. The `connect_matcher` matches only `CONNECT` method requests — when a browser opens a CONNECT tunnel, it is sent to the `internal_tls_api` cluster (which forwards to the internal TLS-terminating listener). The `prefix: "/"` route is the fallback, matching any plain HTTP GET/POST request and sending it directly to the upstream mTLS cluster.

The `CONNECT` route's `connect_config: {}` tells Envoy to **terminate** the CONNECT request — it responds with `200 OK` and then the tunnel is established. Bytes flowing in from the browser after that point are forwarded to the target cluster.

#### HTTP Router filter (lines 79–82)

```yaml
http_filters:
  - name: envoy.filters.http.router
```

This must always be the last filter in the chain. It is the component that actually executes the routing decisions — forwarding requests to the matched cluster. Filters earlier in the chain (auth, rate-limiting, etc.) can modify or reject requests; the router is what sends them onward.

---

### Listener 2: `tls_termination_api` (lines 88–123)

```yaml
- name: tls_termination_api
  internal_listener: {}
```

This is an **internal listener** — it has no network socket. Traffic reaches it only from other parts of Envoy via the `internal_tls_api` cluster (see clusters section). It exists to perform TLS termination on the bytes coming out of the CONNECT tunnel.

When a browser establishes a CONNECT tunnel and then sends a TLS ClientHello (as it always does for HTTPS), those encrypted bytes arrive here. This listener unwraps them.

#### Downstream TLS context (lines 91–103)

```yaml
transport_socket:
  name: envoy.transport_sockets.tls
  typed_config:
    "@type": ...DownstreamTlsContext
    common_tls_context:
      tls_certificates:
        - certificate_chain:
            filename: /etc/envoy/certs/api.example.com.crt
          private_key:
            filename: /etc/envoy/certs/api.example.com.key
      tls_params:
        tls_minimum_protocol_version: TLSv1_2
        tls_maximum_protocol_version: TLSv1_3
```

This is the **server side** of a TLS handshake. Envoy presents `api.example.com.crt` to the browser, the browser verifies it (assuming it has been added to the trust store), and the TLS session is established. After this, Envoy can see the plaintext HTTP inside.

`DownstreamTlsContext` means "I am the TLS server here" — the entity terminating TLS from the client. The key and certificate are mounted from a Kubernetes Secret.

TLS versions are restricted to 1.2 and 1.3. TLS 1.0 and 1.1 are not permitted.

#### HCM inside the internal listener (lines 104–123)

```yaml
filters:
  - name: envoy.filters.network.http_connection_manager
```

Once the browser's TLS is unwrapped, the decrypted bytes are HTTP — so another HCM takes over to parse them and route them.

```yaml
route_config:
  virtual_hosts:
    - name: api
      domains: ["*"]
      routes:
        - match:
            prefix: "/"
          route:
            cluster: cluster_api_example_com
```

The wildcard domain `"*"` matches everything — at this point we already know the request is for `api.example.com` (that was established by the outer listener's routing). The only job here is to forward all requests to `cluster_api_example_com`, which will open the upstream mTLS connection to nginx.

---

## 4. Clusters

Clusters define how Envoy connects to upstream services. Each cluster has an address to connect to, and optionally a transport socket that wraps the connection in TLS.

---

### `internal_tls_api` (lines 128–136)

```yaml
- name: internal_tls_api
  load_assignment:
    endpoints:
      - lb_endpoints:
          - endpoint:
              address:
                envoy_internal_address:
                  server_listener_name: tls_termination_api
```

This cluster has no network address. Instead of a `socket_address`, it has an `envoy_internal_address` pointing to the `tls_termination_api` listener by name. This is what makes the internal listener pattern work — the `forward_proxy` listener routes CONNECT tunnels to this cluster, which delivers the traffic to the `tls_termination_api` internal listener rather than out to the network.

There is no transport socket here because the connection is internal — it never leaves the Envoy process.

---

### `cluster_api_example_com` (lines 139–167)

```yaml
- name: cluster_api_example_com
  type: LOGICAL_DNS
  dns_lookup_family: V4_ONLY
  load_assignment:
    endpoints:
      - lb_endpoints:
          - endpoint:
              address:
                socket_address:
                  address: demo-webserver.default.svc.cluster.local
                  port_value: 443
```

`LOGICAL_DNS` means Envoy resolves the hostname via DNS and connects to the result. It re-resolves periodically but maintains a single connection per resolved address. This is appropriate for Kubernetes in-cluster DNS names where the IP may change if the pod restarts.

`demo-webserver.default.svc.cluster.local` is the fully-qualified DNS name of the nginx Kubernetes Service in the `default` namespace. Envoy connects to port 443.

#### Upstream TLS context (lines 151–167)

```yaml
transport_socket:
  name: envoy.transport_sockets.tls
  typed_config:
    "@type": ...UpstreamTlsContext
    sni: api.example.com
    common_tls_context:
      tls_certificates:
        - certificate_chain:
            filename: /etc/envoy/client-certs/client.crt
          private_key:
            filename: /etc/envoy/client-certs/client.key
      validation_context:
        trusted_ca:
          filename: /etc/envoy/upstream-ca/webserver-ca.crt
      tls_params:
        tls_minimum_protocol_version: TLSv1_2
        tls_maximum_protocol_version: TLSv1_3
```

`UpstreamTlsContext` means "I am the TLS client here" — Envoy is initiating the connection to nginx.

**`sni: api.example.com`** — Sets the TLS SNI (Server Name Indication) field in the ClientHello. This tells nginx which hostname Envoy is connecting to, allowing nginx to select the correct certificate if it serves multiple hostnames. It also means nginx's certificate must be valid for `api.example.com`.

**`tls_certificates`** — This is the **mTLS** part. Envoy presents `client.crt` (CN=scanner_sysid) to nginx during the TLS handshake. nginx is configured to require and verify this certificate. Without this block, the connection would be standard one-way TLS (only nginx authenticates itself); with it, both sides authenticate — mutual TLS.

**`validation_context.trusted_ca`** — Envoy verifies nginx's server certificate against `webserver-ca.crt`. This is the CA that signed nginx's server cert. Without this check, Envoy would accept any certificate from any server, opening the connection to MitM attacks. The file is mounted from the `envoy-upstream-ca` Kubernetes Secret.

---

### `cluster_app_example_com` and `cluster_auth_example_com` (lines 170–229)

These two clusters follow the identical pattern but point to external hostnames (`app.example.com` and `auth.example.com`) rather than the in-cluster demo web server.

The key difference from `cluster_api_example_com` is:

```yaml
validation_context:
  trusted_ca:
    filename: /etc/ssl/certs/ca-certificates.crt
```

These clusters use the system CA bundle rather than a pinned CA. This is appropriate when connecting to public internet services with publicly-trusted certificates, but is a security weakness for internal services (see `security-review.md`, HIGH-5).

---

## Data Flow Summary

```
Browser
  │
  │  CONNECT api.example.com:443 HTTP/1.1   (plaintext → port 3128)
  ▼
[Listener 1: forward_proxy]
  │  matches domains: api.example.com:443
  │  connect_matcher → route to internal_tls_api cluster
  │
  │  200 OK (tunnel established)
  │
  │  <TLS ClientHello from browser>
  ▼
[Cluster: internal_tls_api]
  │  envoy_internal_address → tls_termination_api listener
  ▼
[Listener 2: tls_termination_api]  (internal, no network socket)
  │  DownstreamTlsContext: presents api.example.com.crt to browser
  │  TLS handshake completes; browser's traffic is now decrypted
  │
  │  GET / HTTP/1.1 (plaintext HTTP, now visible to Envoy)
  │  route: prefix "/" → cluster_api_example_com
  ▼
[Cluster: cluster_api_example_com]
  │  DNS: demo-webserver.default.svc.cluster.local:443
  │  UpstreamTlsContext:
  │    - sni: api.example.com
  │    - presents client.crt (CN=scanner_sysid) to nginx   ← mTLS
  │    - verifies nginx cert against webserver-ca.crt
  ▼
nginx (demo-webserver)
  │  ssl_verify_client on → verifies client.crt against lab-ca ✓
  │  serves response
  ▼
Response travels back through the same chain to the browser
```

---

## Key Concepts

| Concept | Where used | What it means |
|---------|-----------|---------------|
| `DownstreamTlsContext` | Listener 2 | Envoy acts as TLS **server** toward the browser |
| `UpstreamTlsContext` | All clusters | Envoy acts as TLS **client** toward the origin |
| `tls_certificates` in upstream | `cluster_api_example_com` | Envoy presents a client certificate — this is the **mTLS** part |
| `validation_context` | All clusters | Envoy verifies the origin's server certificate |
| `connect_matcher` | Listener 1 routes | Matches HTTP `CONNECT` method only |
| `connect_config: {}` | Listener 1 routes | Terminates the CONNECT — Envoy responds 200 and owns the tunnel |
| `envoy_internal_address` | `internal_tls_api` cluster | Routes to an internal listener, not a network socket |
| `internal_listener: {}` | Listener 2 | Creates a virtual listener with no network binding |
| `LOGICAL_DNS` | All real clusters | Resolve hostname via DNS; reconnect if IP changes |
