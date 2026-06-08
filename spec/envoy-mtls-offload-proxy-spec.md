# Envoy mTLS Offload Forward Proxy — Configuration Specification

## Overview

Generate a complete Envoy proxy configuration that acts as an **explicit forward proxy**
whose sole purpose is to **offload mTLS client certificate authentication** to the proxy
on behalf of clients that do not hold a client certificate themselves.

To present a client certificate to the origin, the proxy must own both TLS legs. It
therefore terminates the downstream TLS session from the client and originates a separate
mTLS session upstream. The proxy does **not** inspect or modify HTTP application data —
TLS termination is a structural requirement of mTLS offload, not an inspection feature.

---

## Architecture

```
Client (no client cert, explicit proxy config)
    │
    │  HTTP CONNECT hostname:443   (TLS-encrypted, port 3128)
    │  (TLS using proxy's listener cert — e.g. proxy.example.com)
    ▼
┌──────────────────────────────────────────┐
│               Envoy Proxy                │
│                                          │
│  1. Terminate outer TLS on port 3128     │
│     - present listener cert to client    │
│  2. Accept CONNECT request (plaintext    │
│     inside the outer TLS tunnel)         │
│  3. Terminate inner (downstream) TLS     │
│     - present hostname cert to client    │
│  4. Originate upstream mTLS connection   │
│     - present client cert to origin      │
│     - verify origin server cert          │
│  5. Forward decrypted bytes              │
└──────────────────────────────────────────┘
    │
    │  mTLS (client cert presented, origin cert verified)
    ▼
Origin Server (known hostname)
```

### TLS Session Summary

There are **three distinct TLS sessions** in the complete data path. Each session is independent — the proxy owns all three.

| Session | Leg | Encrypted? | Description |
|---------|-----|-----------|-------------|
| **Session A** | Client → Proxy (port 3128) | **Yes — TLS** | Outer TLS wrapping the CONNECT tunnel. Client authenticates the proxy listener cert. No client certificate required. |
| **Session B** | Client ↔ Proxy (inside Session A) | **Yes — TLS** | Inner TLS for the target hostname. Proxy presents the per-hostname downstream cert. Client may verify against the downstream CA. |
| **Session C** | Proxy → Origin | **Yes — mTLS** | Upstream mutual TLS. Proxy presents its client certificate; origin verifies it. Proxy verifies origin server certificate. |

> **No plaintext anywhere.** Every segment of the data path is TLS-encrypted. The CONNECT
> request itself (Session A) is protected by the outer TLS session on port 3128. The
> payload tunnelled through CONNECT (Session B) is independently TLS-encrypted with a
> hostname-specific certificate. The upstream leg (Session C) is mTLS.

---

## Requirements

### 1. Listener (Outer TLS — Session A)

- Bind on **`0.0.0.0:3128`** (standard explicit proxy port).
- Accept **TLS connections** from the client. The listener itself is TLS-wrapped so that
  the CONNECT request and all subsequent data are always encrypted in transit.
- The listener presents the **same certificate used for downstream TLS (Session B)** — the
  existing wildcard/multi-SAN cert stored at `/etc/envoy/certs/`. The proxy hostname (e.g.
  `proxy.example.com`) must be added as an additional SAN to that certificate so that
  clients can validate the proxy connection. No separate secret or volume mount is needed.
- TLS versions: accept **TLSv1.2 and TLSv1.3** on the listener.
- Use the **`http_connection_manager`** (HCM) network filter.
- Set HCM `codec_type` to `HTTP1` (clients send HTTP/1.1 CONNECT).
- The listener certificate path (same secret as downstream):
  ```
  /etc/envoy/certs/tls.crt   # PEM certificate — must include proxy hostname SAN
  /etc/envoy/certs/tls.key   # PEM private key
  ```

#### Configuring Clients

Clients must be configured with:
- Proxy host: `<proxy-service-hostname>` (or IP)
- Proxy port: `3128`
- Proxy protocol: **HTTPS** (not plain HTTP)
- The listener CA must be trusted by the client (or the listener cert accepted) to avoid
  TLS errors on the proxy connection itself.

### 2. CONNECT Handling

- Enable HTTP CONNECT method support via HCM `upgrade_configs`:
  ```
  upgrade_type: CONNECT
  ```
- The CONNECT request arrives inside Session A (the outer TLS session) and is therefore
  always encrypted on the wire.
- After the proxy accepts the CONNECT request it must **not** forward CONNECT upstream.
  Instead it must terminate the tunnel and begin acting as a TLS endpoint toward the client
  for Session B (the inner/downstream TLS session).
- Use Envoy's **`CONNECT` termination mode** so the proxy itself owns all TLS legs.

### 3. Downstream TLS (Client → Proxy, inner — Session B)

- After accepting the CONNECT for a given hostname, the proxy presents the **pre-generated
  certificate for that hostname** to the client.
- Certificates are stored on disk. The path convention is:
  ```
  /etc/envoy/certs/<hostname>.crt   # PEM certificate (may include chain)
  /etc/envoy/certs/<hostname>.key   # PEM private key
  ```
- TLS versions: accept **TLSv1.2 and TLSv1.3** from the client.
- Use a **`filter_chain_match`** on `server_names` (SNI from the inner TLS ClientHello) to
  select the correct certificate per hostname.
- One filter chain must be defined per known upstream hostname.

### 4. Upstream mTLS (Proxy → Origin — Session C)

The proxy must perform **mutual TLS** on every upstream connection, meaning it both
verifies the origin server's certificate **and** presents its own client certificate for
the origin to verify.

#### 4a. Client Certificate (Proxy authenticates to Origin)

- A single client certificate and private key are used for **all** upstream connections.
- The files are available at:
  ```
  /etc/envoy/client-certs/client.crt   # PEM client certificate (may include chain)
  /etc/envoy/client-certs/client.key   # PEM client private key
  ```
- These paths must be referenced in the `tls_certificates` field of every upstream cluster's
  `envoy.transport_sockets.tls` transport socket configuration.

#### 4b. Origin Server Verification (Proxy verifies Origin)

- Verify the origin server's certificate against the **system CA bundle** at
  `/etc/ssl/certs/ca-certificates.crt`.
- Set `SNI` on each upstream connection to match the requested hostname.
- TLS versions: **TLSv1.2 and TLSv1.3** only.
- Do not add certificate pinning unless explicitly requested.

#### 4c. Upstream Transport Socket

All upstream clusters share the same client cert. Example shape:

```yaml
transport_socket:
  name: envoy.transport_sockets.tls
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
    sni: <hostname>
    common_tls_context:
      tls_certificates:
        - certificate_chain:
            filename: /etc/envoy/client-certs/client.crt
          private_key:
            filename: /etc/envoy/client-certs/client.key
      combined_validation_context:
        default_validation_context:
          trusted_ca:
            filename: /etc/ssl/certs/ca-certificates.crt
      tls_params:
        tls_minimum_protocol_version: TLSv1_2
        tls_maximum_protocol_version: TLSv1_3
```

### 5. Routing

- The HCM route config must match the `CONNECT` authority (hostname:port) and route to the
  correct upstream cluster.
- Each known hostname maps to its own cluster.
- Cluster endpoint is the hostname resolved via DNS (`LOGICAL_DNS` type) on port `443`.

### 6. Known Upstream Hostnames

The following hostnames must be supported. Add one filter chain and one cluster for each:

```
api.example.com
app.example.com
auth.example.com
```

> **Note to implementer:** Replace this list with the actual target hostnames before
> generating the config. The structure must be repeated for each entry. The client cert
> paths in section 4a are shared across all clusters and do not change per hostname.

### 7. Access Logging

- Log all requests to **stdout** using the **file access log** with the following format:
  ```
  [%START_TIME%] "%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%"
  %RESPONSE_CODE% %BYTES_SENT% "%REQ(HOST)%" "%REQ(USER-AGENT)%"
  ```

### 8. Admin Interface

- Enable the Envoy admin interface on **`127.0.0.1:9901`** for health checking and stats.
- Do **not** expose admin on `0.0.0.0`.

---

## Configuration Format

- Output a **single `envoy.yaml`** file using the **Envoy v3 xDS API** (`@type` references
  under `envoy.config.listener.v3.*` etc.).
- Use **static configuration only** (`static_resources`) — no xDS management server.
- Target **Envoy 1.38+**.

---

## Certificate Preparation (Out of Scope for Config)

The Envoy config itself does not generate certificates. The following must be in place
before starting Envoy.

### Downstream / Listener Certificate (for Sessions A and B — shared)

The same certificate is used for both the outer listener TLS (Session A) and the inner
per-hostname TLS (Session B). There is no separate listener certificate secret.

The **proxy hostname** (e.g. `proxy.example.com`) must be included as a SAN on the
downstream certificate alongside all the upstream target hostnames. The certificate is
already mounted at `/etc/envoy/certs/` — the only change required is to regenerate it
with the additional SAN.

Clients must trust the CA that signed this certificate (or trust the cert directly) in
order to validate both the proxy connection and the per-hostname connections. In a
corporate PKI environment this would typically be the internal CA already distributed
to clients.

### Per-Hostname Downstream Certificates (for Session B — inner TLS)

One certificate per known upstream hostname, placed at the paths in section 3.

Where a wildcard or multi-SAN certificate is used (the recommended approach), a single
cert covers all downstream hostnames **and** the proxy listener hostname. The SAN list
must include every target hostname plus the proxy's own hostname:

```bash
# Example: single cert covering all downstream hostnames and the proxy hostname
openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes \
  -keyout /etc/envoy/certs/tls.key \
  -out    /etc/envoy/certs/tls.crt \
  -subj   "/CN=proxy.example.com" \
  -addext "subjectAltName=DNS:proxy.example.com,DNS:api.example.com,DNS:app.example.com,DNS:auth.example.com"
```

The client must trust the CA (or this cert directly) to avoid TLS warnings on both the
proxy connection and the per-hostname connections presented by the proxy.

### Client Certificate for mTLS (for Session C — upstream mTLS)

A single client certificate used by Envoy to authenticate to all upstream origins. This
certificate must be issued or trusted by the CA that each origin server uses for client
authentication:

```bash
# Generate a key and CSR, then have it signed by the appropriate CA
openssl req -newkey rsa:4096 -nodes \
  -keyout /etc/envoy/client-certs/client.key \
  -out    /etc/envoy/client-certs/client.csr \
  -subj   "/CN=envoy-proxy-client"

# Sign with your CA (example using a local CA key)
openssl x509 -req -days 365 -sha256 \
  -in    /etc/envoy/client-certs/client.csr \
  -CA    /path/to/ca.crt \
  -CAkey /path/to/ca.key \
  -CAcreateserial \
  -out   /etc/envoy/client-certs/client.crt
```

Ensure `/etc/envoy/client-certs/` has permissions readable only by the Envoy process.

---

## What to Generate

Produce a complete, working `envoy.yaml` that:

1. Defines a single listener on `0.0.0.0:3128` **with TLS enabled** (Session A), presenting
   the downstream certificate from `/etc/envoy/certs/tls.{crt,key}` (the same cert used
   for Session B — the proxy hostname must be included as a SAN on that certificate).
2. Contains one `filter_chain` per known hostname (Session B), each with:
   - A `filter_chain_match.server_names` entry for that hostname.
   - An `envoy.transport_sockets.tls` downstream TLS config loading the correct cert/key
     for that hostname.
   - An HCM with CONNECT upgrade enabled and a route to the corresponding upstream cluster.
3. Defines one `LOGICAL_DNS` cluster per known hostname with:
   - An mTLS upstream transport socket (Session C) presenting
     `/etc/envoy/client-certs/client.crt` and `/etc/envoy/client-certs/client.key`.
   - Origin certificate verification against the system CA bundle.
   - Correct SNI set to the hostname.
4. Includes access logging to stdout.
5. Includes the admin interface on `127.0.0.1:9901`.

Do not omit any required fields. The config must pass `envoy --mode validate`.

---

## Implementation Notes (Two-Listener Pattern)

The implemented Helm chart uses Envoy's **internal listener** feature to split the TLS
processing across two listeners:

- **Listener 1 (`forward_proxy`, port 3128):** Accepts the outer TLS connection (Session A)
  and the CONNECT request. Routes the CONNECT tunnel to an internal cluster.
- **Listener 2 (`tls_termination_wildcard`, internal):** Terminates the inner hostname TLS
  (Session B) and forwards to the dynamic forward proxy cluster (Session C).

This is the correct architectural pattern and should be preserved when adding the outer TLS
(Session A) to Listener 1.

The `bootstrap_extensions` block enabling `envoy.bootstrap.internal_listener` is required
for this pattern to work.

---

## Helm Chart Changes Required

Adding TLS to the port-3128 listener requires the following chart changes:

1. **No new secret required.** The existing `envoy-downstream-certs` secret is reused for
   the listener TLS. The only prerequisite is that the certificate in that secret is
   regenerated to include the proxy hostname as an additional SAN.
2. **Updated `envoy-configmap.yaml`:** Add a `transport_socket` with a
   `DownstreamTlsContext` to Listener 1 (`forward_proxy`), referencing the existing
   downstream cert files at `/etc/envoy/certs/tls.{crt,key}`.
3. **Updated `self-managed-certs.md`:** Update the cert generation step to include the
   proxy hostname SAN alongside the upstream target hostnames.
4. **Client proxy configuration:** Update all clients to use `https://` (not `http://`) as
   the proxy scheme, or the `HTTPS_PROXY` environment variable.

---

## Out of Scope

The following are explicitly **not** required in this config:

- HTTP inspection, filtering, or modification of application data.
- Dynamic certificate generation or an internal CA.
- Authentication of the client to the proxy (separate concern — see security-review.md CRIT-2).
- HTTP/2 or HTTP/3 support.
- Dynamic xDS resource discovery.
- Rate limiting or circuit breaking.
- Per-hostname client certificates (one shared client cert is sufficient).
