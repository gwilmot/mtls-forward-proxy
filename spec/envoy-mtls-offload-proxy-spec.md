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
    │  HTTP CONNECT hostname:443 HTTP/1.1   (plaintext, port 3128)
    ▼
┌──────────────────────────────────────────┐
│               Envoy Proxy                │
│                                          │
│  1. Accept CONNECT request               │
│  2. Terminate downstream TLS             │
│     - present hostname cert to client    │
│  3. Originate upstream mTLS connection   │
│     - present client cert to origin      │
│     - verify origin server cert          │
│  4. Forward decrypted bytes              │
└──────────────────────────────────────────┘
    │
    │  mTLS (client cert presented, origin cert verified)
    ▼
Origin Server (known hostname)
```

The client's TLS session terminates at the proxy. The proxy re-originates a separate mTLS
session to the origin. These are two independent TLS sessions — the proxy must own both in
order to inject the client certificate on the upstream leg.

---

## Requirements

### 1. Listener

- Bind on **`0.0.0.0:3128`** (standard explicit proxy port).
- Accept plain TCP connections from the client (no TLS on the inbound listener itself — the
  client connects in plaintext to send its CONNECT request first).
- Use the **`http_connection_manager`** (HCM) network filter.
- Set HCM `codec_type` to `HTTP1` (clients send HTTP/1.1 CONNECT).

### 2. CONNECT Handling

- Enable HTTP CONNECT method support via HCM `upgrade_configs`:
  ```
  upgrade_type: CONNECT
  ```
- After the proxy accepts the CONNECT request it must **not** forward CONNECT upstream.
  Instead it must terminate the tunnel and begin acting as a TLS endpoint toward the client.
- Use Envoy's **`CONNECT` termination mode** so the proxy itself owns both TLS legs.

### 3. Downstream TLS (Client → Proxy)

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

### 4. Upstream mTLS (Proxy → Origin)

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
- Target **Envoy 1.29+**.

---

## Certificate Preparation (Out of Scope for Config)

The Envoy config itself does not generate certificates. The following must be in place
before starting Envoy.

### Per-Hostname Downstream Certificates

One certificate per known upstream hostname, placed at the paths in section 3:

```bash
# Repeat for each hostname
openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes \
  -keyout /etc/envoy/certs/api.example.com.key \
  -out    /etc/envoy/certs/api.example.com.crt \
  -subj   "/CN=api.example.com" \
  -addext "subjectAltName=DNS:api.example.com"
```

The client must trust the CA (or these self-signed certs directly) to avoid TLS warnings.

### Client Certificate for mTLS

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

1. Defines a single listener on `0.0.0.0:3128`.
2. Contains one `filter_chain` per known hostname, each with:
   - A `filter_chain_match.server_names` entry for that hostname.
   - An `envoy.transport_sockets.tls` downstream TLS config loading the correct cert/key
     for that hostname.
   - An HCM with CONNECT upgrade enabled and a route to the corresponding upstream cluster.
3. Defines one `LOGICAL_DNS` cluster per known hostname with:
   - An mTLS upstream transport socket presenting `/etc/envoy/client-certs/client.crt`
     and `/etc/envoy/client-certs/client.key`.
   - Origin certificate verification against the system CA bundle.
   - Correct SNI set to the hostname.
4. Includes access logging to stdout.
5. Includes the admin interface on `127.0.0.1:9901`.

Do not omit any required fields. The config must pass `envoy --mode validate`.

---

## Out of Scope

The following are explicitly **not** required in this config:

- HTTP inspection, filtering, or modification of application data.
- Dynamic certificate generation or an internal CA.
- Authentication of the client to the proxy.
- HTTP/2 or HTTP/3 support.
- Dynamic xDS resource discovery.
- Rate limiting or circuit breaking.
- Per-hostname client certificates (one shared client cert is sufficient).
