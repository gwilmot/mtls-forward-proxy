# Security Review: Envoy mTLS Offload Forward Proxy

**Scope:** Envoy mTLS forward proxy + nginx demo web server + Kubernetes manifests
**Date:** 2026-05-17
**Context:** Lab environment — findings are rated against production readiness

---

## Summary Table

| ID | Severity | Area | One-line fix |
|----|----------|------|--------------|
| CRIT-1 | CRITICAL | Key management | Revoke all keys; never write private keys to working tree; use K8s Secrets only |
| CRIT-2 | CRITICAL | Access control | Add RBAC/authn filter on port 3128 downstream listener |
| CRIT-3 | CRITICAL | Crypto/architecture | Protect interception CA private key with HSM/Vault; never put it on disk |
| CRIT-4 | CRITICAL | K8s security | Add NetworkPolicy restricting proxy and web server ingress |
| HIGH-1 | HIGH | Supply chain | Pin envoy image to immutable digest |
| HIGH-2 | HIGH | K8s security | Add securityContext to demo-webserver (runAsNonRoot, readOnlyRootFilesystem, drop ALL) |
| HIGH-3 | HIGH | Auth | Remove port 80 plaintext listener from nginx.conf |
| HIGH-4 | HIGH | Crypto | Issue separate client certs per upstream cluster |
| HIGH-5 | HIGH | Crypto | Pin app/auth upstream clusters to their own CA, not the system bundle |
| MED-1 | MEDIUM | Crypto | Restrict cipher suites to ECDHE+AEAD suites in both Envoy and nginx |
| MED-2 | MEDIUM | Exposure | Remove admin containerPort 9901 from Deployment spec |
| MED-3 | MEDIUM | K8s security | Change Secret volume defaultMode from 0444 to 0400 |
| MED-4 | MEDIUM | K8s security | Create dedicated ServiceAccounts; set automountServiceAccountToken: false |
| MED-5 | MEDIUM | Cert lifecycle | Automate cert rotation via cert-manager; monitor expiry |
| MED-6 | MEDIUM | Config drift | Reconcile local nginx.conf with ConfigMap; single source of truth |
| MED-7 | MEDIUM | Observability | Replace tcpSocket probes with HTTP/TLS application-layer probes |
| LOW-1 | LOW | gitignore | Add *.key/*.crt patterns to subdirectory .gitignore files |
| LOW-2 | LOW | gitignore | Broaden secret manifest exclusion patterns; add pre-commit scanning |
| LOW-3 | LOW | K8s security | Move workloads to a dedicated namespace |
| LOW-4 | LOW | Supply chain | Pin nginx image to immutable digest |
| LOW-5 | LOW | Logging | Switch to structured JSON access log to prevent log injection |
| LOW-6 | LOW | Architecture | Document threat model for TLS interception pattern |

---

## CRITICAL

---

### CRIT-1 — Private keys present in the working tree

**Files:**
- `envoy-proxy/certs/api.example.com.key`
- `envoy-proxy/client-certs/client.key`
- `web-server/certs/server.key`

All three private keys exist on disk inside the project tree. The `.gitignore` files use pattern-based exclusions (`*.key`, `certs/`) but pattern-based gitignore silently fails once a file has been staged — and the patterns in the subdirectory `.gitignore` files do not include `*.key` or `*.crt` directly (they rely on directory-level ignores). Any key placed outside those named directories would not be caught.

Additionally, if any key was committed before the `.gitignore` was fully populated it will remain in git history permanently even after deletion from the working tree.

**Recommendation:** Treat all three key pairs as potentially compromised. Revoke and regenerate all certificates and keys. Audit git history with `git log --all --full-history -- "*.key"`. Use BFG Repo Cleaner or `git filter-repo` to purge history if keys were ever committed. Store private keys exclusively in Kubernetes Secrets or a secrets manager (Vault, AWS Secrets Manager) and never write them to the working tree. Add a pre-commit hook (e.g., `git-secrets`, `truffleHog`, `detect-secrets`) to prevent future leaks.

---

### CRIT-2 — Envoy forward proxy has no client authentication — effectively an open proxy

**File:** `envoy-proxy/envoy.yaml`, listener on `0.0.0.0:3128`

The listener enforces routing by `Host`/CONNECT destination domain only. There is no client authentication: no downstream mTLS, no `Proxy-Authorization` header check, no RBAC filter, no ExtAuthz integration. Any pod or workload that can reach the proxy service at `envoy-mtls-proxy:3128` can have the proxy present the mTLS client certificate to backend services on its behalf. In production the proxy's client certificate may grant privileged access to upstream APIs, meaning any unauthenticated caller inherits those privileges.

**Recommendation:** Add one or more of:
- Downstream mTLS on port 3128, requiring callers to present their own certificate
- Envoy `envoy.filters.http.rbac` filter restricting by source IP or principal
- `Proxy-Authorization` basic auth via `envoy.filters.http.basic_auth`
- An ExtAuthz sidecar for policy-based authorisation

Pair with NetworkPolicy (see CRIT-4).

---

### CRIT-3 — Interception CA private key on disk creates unlimited MitM capability

**File:** `envoy-proxy/certs/api.example.com.crt` (and associated `.key`)

Listener 2 (`tls_termination_api`) presents a self-signed certificate for `api.example.com` to the browser to decrypt its TLS session. This is the intended forward-proxy pattern, but it carries a critical operational risk: the CA that signed this certificate must be trusted by client browsers. If the CA's private key is stolen, an attacker can issue certificates for any domain and perform MitM attacks against any HTTPS session from those clients — far beyond this lab's scope.

The `api.example.com.crt` certificate carries `CA:TRUE` in its `basicConstraints`, meaning it can itself issue sub-certificates for other hostnames.

**Recommendation:** In production the downstream interception CA must be a dedicated, offline, hardware-protected CA (HSM or cloud KMS). Its private key must never touch disk on the proxy pod. Use cert-manager with Vault PKI or a cloud-native CA service. Scope the trust anchor installation to only the managed clients that require proxy access, and never install it in a browser's global trust store.

---

### CRIT-4 — No Kubernetes NetworkPolicy — proxy reachable by any pod in the cluster

**Files:** No `NetworkPolicy` manifest exists anywhere in the project.

Without a NetworkPolicy, every pod in every namespace can send traffic to `envoy-mtls-proxy:3128`. Combined with the lack of proxy authentication (CRIT-2), any workload in the cluster can use the mTLS client credential. Similarly, `demo-webserver:443` is reachable by any pod, bypassing the intended access model where only the proxy presents the client cert.

**Recommendation:** Create NetworkPolicy resources for both components:

```yaml
# Allow ingress to proxy only from labelled clients
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-proxy-clients
spec:
  podSelector:
    matchLabels:
      app: envoy-mtls-proxy
  ingress:
    - from:
        - podSelector:
            matchLabels:
              proxy-client: "true"
      ports:
        - port: 3128
---
# Allow ingress to web server only from proxy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-webserver-from-proxy
spec:
  podSelector:
    matchLabels:
      app: demo-webserver
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: envoy-mtls-proxy
      ports:
        - port: 443
```

---

## HIGH

---

### HIGH-1 — Floating image tag `envoyproxy/envoy:v1.29-latest`

**File:** `envoy-proxy/k8s/deployment.yaml`, line 20

The tag `v1.29-latest` is mutable. It can silently resolve to a different image at any pod restart, making the deployment non-deterministic and bypassing image scanning results.

**Recommendation:** Pin to an immutable digest: `envoyproxy/envoy:v1.29.9@sha256:<digest>`. Enforce this with an admission controller (OPA Gatekeeper, Kyverno).

---

### HIGH-2 — nginx web server runs as root with no securityContext

**File:** `web-server/k8s/deployment.yaml`

The `demo-webserver` deployment has no `securityContext` at pod or container level. The nginx:1.27 master process runs as root (UID 0) by default with a writable filesystem. Contrast this with the well-hardened Envoy deployment which sets `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, and drops all capabilities.

**Recommendation:**

```yaml
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 101  # nginx standard non-root UID
  capabilities:
    drop: ["ALL"]
```

Mount writable `emptyDir` volumes for `/tmp`, `/var/cache/nginx`, and `/var/run` which nginx requires for operation.

---

### HIGH-3 — Unauthenticated plaintext HTTP listener on nginx port 80

**File:** `web-server/nginx.conf`, lines 3–14

The local `nginx.conf` exposes a plain HTTP listener on port 80 that returns 200 to any request with no authentication. There is no enforcement that traffic arrives only from the Envoy proxy — any pod reaching this port bypasses mTLS entirely. This also represents configuration drift: the Kubernetes ConfigMap does not include this listener, creating a discrepancy between local and deployed configuration.

**Recommendation:** Remove the port 80 listener from `nginx.conf` entirely. All traffic to the web server should arrive on port 443 with mTLS enforced. See also MED-6.

---

### HIGH-4 — Single shared mTLS client certificate for all upstream clusters

**File:** `envoy-proxy/envoy.yaml`, upstream cluster `tls_certificates` blocks

All three upstream clusters use the identical `client.crt` / `client.key` pair. If the certificate is revoked, all upstream connectivity fails simultaneously. If any upstream service is compromised and exfiltrates the client certificate, it gains the same identity as all other upstreams. `auth.example.com` receives the same identity as the API service, potentially granting unintended access.

**Recommendation:** Issue separate client certificates per upstream cluster with distinct Subject CNs. Store each in its own Kubernetes Secret. This enables per-service revocation and limits blast radius.

---

### HIGH-5 — `app.example.com` and `auth.example.com` clusters trust the system CA bundle

**File:** `envoy-proxy/envoy.yaml`, `validation_context.trusted_ca` for `cluster_app_example_com` and `cluster_auth_example_com`

```yaml
trusted_ca:
  filename: /etc/ssl/certs/ca-certificates.crt
```

The `api.example.com` cluster correctly uses a pinned project-specific CA. The other two clusters trust the entire system CA store, meaning the proxy will accept any certificate signed by any public CA — including a mis-issued or attacker-controlled certificate — for those services.

**Recommendation:** Pin each upstream cluster to its own dedicated CA certificate, exactly as done for `cluster_api_example_com`. Never use the system CA bundle for internal service-to-service mTLS.

---

## MEDIUM

---

### MED-1 — No cipher suite restriction on TLS contexts

**Files:** `envoy-proxy/envoy.yaml` (`tls_params` blocks), `web-server/nginx.conf` (`ssl_protocols`)

Neither Envoy nor nginx restrict cipher suites. For TLS 1.2 connections, negotiated suites are left to implementation defaults which may include suites lacking forward secrecy (e.g., `TLS_RSA_*`).

**Recommendation:** For Envoy, add `cipher_suites` in each `tls_params` block restricted to ECDHE-based suites. For nginx, add:
```nginx
ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
ssl_prefer_server_ciphers on;
```
If all clients support TLS 1.3, consider restricting to TLS 1.3 only and eliminating TLS 1.2 cipher concerns entirely.

---

### MED-2 — Admin interface containerPort 9901 declared in Deployment

**File:** `envoy-proxy/k8s/deployment.yaml`, line 31

The Envoy admin interface is correctly bound to `127.0.0.1:9901` and not externally reachable. However, declaring it as a named `containerPort` makes it visible to tooling and service meshes, and implies external consumption. If the bind address were changed to `0.0.0.0`, the declared port would already be wired for exposure. The admin interface provides hot-restart, configuration dumps, and traffic manipulation.

**Recommendation:** Remove the `admin` containerPort declaration. Expose admin functionality only via a dedicated observability sidecar if needed externally.

---

### MED-3 — Secret volumes mounted world-readable (`defaultMode: 0444`)

**Files:** `envoy-proxy/k8s/deployment.yaml`, `web-server/k8s/deployment.yaml`

Secret volumes including private keys are mounted with `defaultMode: 0444`, making them readable by all UIDs in the container. Any compromised dependency running in the same container can read private key material without privilege escalation.

**Recommendation:** Change `defaultMode` to `0400`. The Envoy container runs as UID 1001 and the nginx container as UID 101 — files owned by root and mode 0400 are still readable by the process when mounted via Kubernetes Secret volumes (the kubelet sets ownership correctly).

---

### MED-4 — No dedicated ServiceAccounts; API token automounted

**Files:** No `ServiceAccount` manifest in either `k8s/` directory.

Both pods use the `default` ServiceAccount and automount a Kubernetes API token. Neither workload requires Kubernetes API access. A compromised container could use the mounted token to query or modify cluster resources.

**Recommendation:** Create dedicated ServiceAccounts for each Deployment and set `automountServiceAccountToken: false` in both Deployment specs.

---

### MED-5 — No certificate rotation mechanism; one-year validity with no expiry monitoring

All certificates are valid for 365 days with no rotation automation, no OCSP/CRL infrastructure, and no expiry monitoring. They will silently expire causing outages without warning.

**Recommendation:** Use cert-manager with an appropriate Issuer for automated issuance and rotation. Set certificate lifetimes to 90 days or less. Add expiry monitoring via Prometheus blackbox exporter or cert-manager's built-in metrics.

---

### MED-6 — Configuration drift between `nginx.conf` and the Kubernetes ConfigMap

**Files:** `web-server/nginx.conf` vs `web-server/k8s/configmap.yaml`

The local `nginx.conf` includes a port 80 listener; the ConfigMap does not. These files are maintained independently with no automated reconciliation. This is a recurring source of security defects where what developers test locally differs from what runs in production.

**Recommendation:** Treat `nginx.conf` as the single source of truth. Generate the ConfigMap from it: `kubectl create configmap nginx-config --from-file=nginx.conf --dry-run=client -o yaml`. Validate against drift in CI.

---

### MED-7 — TCP socket probes provide no application-layer health validation

**Files:** `envoy-proxy/k8s/deployment.yaml`, `web-server/k8s/deployment.yaml`

TCP probes confirm a port is open but not that the application is serving correctly. A pod where a TLS context has failed to load due to a missing secret would still pass the probe, masking certificate-load failures that silently degrade mTLS enforcement.

**Recommendation:** For Envoy, use `httpGet` on `/ready` (port 9901, loopback only — use an exec probe if needed). For nginx, validate via TLS connection test or application-layer HTTP check.

---

## LOW / Informational

---

### LOW-1 — `.gitignore` coverage gaps in subdirectories

**Files:** `envoy-proxy/.gitignore`, `web-server/.gitignore`

The subdirectory `.gitignore` files use directory-level ignores (`certs/`, `client-certs/`) but do not include `*.key` or `*.crt` as standalone patterns. A key placed directly in `envoy-proxy/` (outside the named directories) would not be excluded. The root-level `*.key`/`*.crt` patterns do not cascade into subdirectories that have their own `.gitignore` files.

**Recommendation:** Add `*.key`, `*.crt`, `*.csr`, `*.pem`, `*.p12`, `*.srl` to both `envoy-proxy/.gitignore` and `web-server/.gitignore`.

---

### LOW-2 — Secret manifest exclusion relies on fragile naming convention

**Files:** All `.gitignore` files exclude `k8s/secret-*.yaml`

A secret manifest named `credentials.yaml` or `certs-secret.yaml` would not be excluded. There is no enforcement of the `secret-` prefix convention.

**Recommendation:** Broaden to `k8s/*secret*.yaml` and `k8s/*cred*.yaml`, or use a pre-commit hook that scans all committed YAML files for base64-encoded PEM data.

---

### LOW-3 — Both deployments in the `default` namespace

**Files:** `envoy-proxy/k8s/deployment.yaml`, `web-server/k8s/deployment.yaml`

Using `default` provides no isolation boundary and makes NetworkPolicy and RBAC scoping harder.

**Recommendation:** Deploy to a dedicated namespace (e.g., `mtls-proxy`) with ResourceQuotas and LimitRanges applied.

---

### LOW-4 — `nginx:1.27` image tag not pinned to a digest

**File:** `web-server/k8s/deployment.yaml`

Same class of issue as HIGH-1. The `1.27` tag is less volatile than `latest` but is still mutable.

**Recommendation:** Pin to `nginx:1.27@sha256:<digest>`.

---

### LOW-5 — Access log `User-Agent` field is injectable

**File:** `envoy-proxy/envoy.yaml`, access log `text_format`

```
"%REQ(USER-AGENT)%"
```

The `User-Agent` value is written directly into a text log format string. A malicious client can inject newlines or ANSI escape sequences, potentially corrupting log output or exploiting log ingestion systems.

**Recommendation:** Switch to Envoy's structured JSON access log format, which encodes header values as JSON strings preventing injection.

---

### LOW-6 — Architecture: TLS interception bypasses end-to-end encryption by design

This is an architectural observation rather than a misconfiguration. The design terminates the browser's TLS session inside the cluster, which means:
- Confidentiality of browser-to-API communication depends entirely on cluster security, not end-to-end cryptography
- Any compromise of the proxy pod exposes all decrypted traffic in flight
- The browser's PKI trust in `api.example.com` is satisfied by a proxy-controlled certificate, not the real server's certificate

**Recommendation:** Before production deployment, explicitly document the threat model: who are the trusted parties, what is the trust boundary, and why terminating TLS at the proxy is acceptable. Ensure this decision is reviewed by a security architect and communicated to stakeholders whose data transits the proxy.
