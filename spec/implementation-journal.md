# Implementation Journal: Envoy mTLS Offload Proxy — Spec to Working Lab

## 1. Read the spec
Parsed the spec to understand the architecture: explicit forward proxy on port 3128, CONNECT termination, downstream TLS toward the browser, upstream mTLS to the origin presenting a client certificate.

## 2. Generated certificates
Created four sets of certs using openssl:
- **Lab CA** → signed the **client cert** (CN=scanner_sysid) that Envoy presents to upstream origins
- **Per-hostname self-signed certs** for `api.example.com`, `app.example.com`, `auth.example.com` — presented by Envoy to the browser during the inner TLS handshake
- **Webserver CA** → signed the **nginx server cert** — Envoy verifies this when connecting upstream

## 3. Wrote the Envoy config and K8s manifests
First attempt followed the spec literally — one filter chain per hostname matched on SNI, with downstream TLS. Hit a series of config errors iterating to get Envoy to start:
- `combined_validation_context` → had to change to plain `validation_context` (v1.29 schema change)
- Secret `defaultMode: 0400` → changed to `0444` (non-root Envoy process couldn't read mounted certs)
- Liveness/readiness probes used HTTP to `127.0.0.1:9901` (admin) → kubelet can't reach loopback inside a pod, switched to TCP socket probe on port 3128
- `wget` not in Envoy image → TCP socket probes
- ConfigMap had embedded YAML indentation drift → switched to generating ConfigMap directly from `envoy.yaml` with `kubectl create configmap --from-file`

## 4. Deployed the demo web server
Created nginx with mTLS on port 443 — `ssl_verify_client on` requiring the client cert be signed by the lab CA. Hit:
- `nginx:1.25-alpine` image tag no longer existed → switched to `nginx:1.27`, pre-pulled with `docker pull` to prime the kind node cache

## 5. Wired Envoy upstream to the web server
Updated the `api.example.com` cluster to point at `demo-webserver.default.svc.cluster.local:443` with the webserver CA for verification. Added a new `envoy-upstream-ca` secret and volume mount.

## 6. Fought the CONNECT tunnel architecture
This was the main iterative battle. Several wrong approaches:
- **First attempt**: SNI-based filter chain matching on the outer listener — CONNECT was rejected because SNI isn't available before CONNECT is unwrapped
- **Second attempt**: Single HCM with `upgrade_configs: CONNECT` and routes per hostname — CONNECT tunnel established but client's inner TLS (from browser) passed raw into the tunnel, causing double-TLS at nginx
- **Third attempt**: `allow_connect: true` in `http_protocol_options` — invalid field in v1.29, crashed Envoy
- **Final working architecture**: Two-listener pattern using Envoy's **internal listener**:
  1. Outer HCM listener on 3128 accepts CONNECT, routes the tunnel to an internal cluster
  2. Internal cluster points to an **internal listener** (not a TCP socket)
  3. Internal listener terminates the browser's inner TLS using the hostname cert, then routes plain HTTP to the upstream mTLS cluster
  4. Upstream cluster does mTLS to nginx, presenting the client cert
  - Required adding `envoy.bootstrap.internal_listener` bootstrap extension — without it Envoy rejected the internal listener with "registry not initialized"

## 7. Browser access
- **kind** (not Docker Desktop's built-in K8s) — node runs inside Docker so NodePort and LoadBalancer don't reach the Mac host. `kubectl port-forward` is the only reliable method
- Added `api.example.com` to `/etc/hosts` pointing to `127.0.0.1`
- Installed the self-signed downstream cert into the Mac System keychain and login keychain for Chrome
- Tested with `curl --proxy http://localhost:3128 --insecure https://api.example.com/` to confirm before the browser

## 8. Published to GitHub
- Ensured `.gitignore` at root and in each component directory excluded all `certs/`, `client-certs/`, `*.key`, `*.crt`, and `k8s/secret-*.yaml` files
- Wrote a full README with cert generation commands, secret creation, deploy steps, and trust store instructions
- `git init`, committed 13 files (no certs), `gh repo create --private` pushed to `github.com/gwilmot/mtls-forward-proxy`

## Key lessons learned

| Problem | Root cause | Fix |
|---------|-----------|-----|
| CONNECT rejected | `upgrade_configs` alone insufficient | `allow_connect` doesn't exist in v1.29 — `upgrade_configs: CONNECT` is enough but routing must match |
| Double-TLS | Browser sends TLS into CONNECT tunnel, nginx gets TLS-in-TLS | Internal listener to terminate browser's TLS before routing upstream |
| Browser "not secure" | Self-signed cert needs trust store entry | `security add-trusted-cert` to both System and login keychains |
| Probes failing | Admin on `127.0.0.1` unreachable by kubelet | TCP socket probe on port 3128 |
| ConfigMap YAML drift | Embedded YAML in ConfigMap gets mangled by indentation | Generate ConfigMap directly from the source file |
