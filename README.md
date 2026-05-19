# mTLS Forward Proxy — Lab

An Envoy-based explicit forward proxy that offloads mTLS client certificate authentication on behalf of clients. The client (browser or curl) holds no certificate — Envoy intercepts the TLS session and opens a fresh mTLS connection to the backend, injecting a client certificate.

## Architecture

```
Browser / curl (no client cert)
    │
    │  CONNECT docs.example.com:443   (plaintext, port 3128)
    ▼
┌─────────────────────────────────────────────────────────┐
│                    Envoy Proxy (K8s)                    │
│                                                         │
│  1. Accept CONNECT — wildcard cert (*.example.com)      │
│  2. Terminate downstream TLS                            │
│  3. Resolve upstream via DNS (dynamic forward proxy)    │
│  4. Open mTLS connection presenting CN=scanner_sysid    │
│  5. Bridge decrypted bytes back to client               │
└─────────────────────────────────────────────────────────┘
    │
    │  mTLS — client cert verified by backend
    ▼
Backend nginx pod (any *.example.com hostname)
    - requires client cert signed by lab CA
    - responds with client CN in body
```

## Repo layout

| Path | Description |
|------|-------------|
| `charts/mtls-forward-proxy/` | Helm chart — the only supported way to deploy |
| `spec/` | Architecture spec, config explainer, security review, tech selection |

## Prerequisites

- Kubernetes 1.24+ (tested with [kind](https://kind.sigs.k8s.io/) via Docker Desktop)
- Helm 3.x
- kubectl configured against your cluster
- cert-manager v1.17+ installed in the cluster
- `openssl` is **not required** — all keys are generated in-cluster by cert-manager

## Quick start

### 1. Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.2/cert-manager.yaml
kubectl rollout status deployment/cert-manager deployment/cert-manager-cainjector deployment/cert-manager-webhook \
  -n cert-manager --timeout=120s
```

### 2. Bootstrap the CA keypairs entirely in-cluster

No openssl. No keys on disk. cert-manager generates the CA keypairs inside the cluster and writes them directly to secrets.

```bash
kubectl apply -f - <<'EOF'
# Step 1 — bootstrapper that signs only the CA certs
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
# Step 2 — Lab CA keypair, generated in-cluster, never touches disk
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: lab-ca
  namespace: cert-manager
spec:
  secretName: cert-manager-lab-ca-new
  commonName: lab-ca
  isCA: true
  privateKey:
    algorithm: RSA
    size: 4096
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  duration: 87600h
---
# Step 3 — Webserver CA keypair, generated in-cluster, never touches disk
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: webserver-ca
  namespace: cert-manager
spec:
  secretName: cert-manager-webserver-ca-new
  commonName: webserver-ca
  isCA: true
  privateKey:
    algorithm: RSA
    size: 4096
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  duration: 87600h
EOF

kubectl get certificate -n cert-manager   # both should show READY=True
```

### 3. Create the ClusterIssuers and upstream CA trust anchor

```bash
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: lab-ca-issuer
spec:
  ca:
    secretName: cert-manager-lab-ca-new
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: webserver-ca-issuer
spec:
  ca:
    secretName: cert-manager-webserver-ca-new
EOF

kubectl get clusterissuer   # all three should show READY=True
```

```bash
# Upstream CA trust anchor — Envoy uses this to verify backend server certs
# Extract the webserver CA cert from the in-cluster secret (no private key)
kubectl create secret generic envoy-upstream-ca \
  --from-literal=tls.crt="$(kubectl get secret cert-manager-webserver-ca-new \
    -n cert-manager -o jsonpath='{.data.tls\.crt}' | base64 -d)"
```

### 4. Issue certificates via cert-manager

```bash
kubectl apply -f - <<'EOF'
# Envoy wildcard downstream cert (browser-facing)
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: envoy-downstream-wildcard
  namespace: default
spec:
  secretName: envoy-downstream-certs
  commonName: "*.example.com"
  dnsNames: ["*.example.com"]
  issuerRef:
    name: lab-ca-issuer
    kind: ClusterIssuer
  duration: 87600h
  renewBefore: 720h
---
# Envoy client cert (presented to every upstream backend)
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: envoy-client-cert
  namespace: default
spec:
  secretName: envoy-client-certs
  commonName: scanner_sysid
  subject:
    organizations: [lab]
  usages: [client auth]
  issuerRef:
    name: lab-ca-issuer
    kind: ClusterIssuer
  duration: 87600h
  renewBefore: 720h
---
# Demo backend server cert
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-webserver-cert
  namespace: default
spec:
  secretName: nginx-certs
  commonName: api.example.com
  dnsNames:
    - api.example.com
    - mtls-webserver.default.svc.cluster.local
  issuerRef:
    name: webserver-ca-issuer
    kind: ClusterIssuer
  duration: 87600h
  renewBefore: 720h
EOF

kubectl get certificates   # all should show READY=True within ~10s
```

> **Adding a new backend:** create one `Certificate` resource with the public hostname and K8s service FQDN as SANs, using `webserver-ca-issuer`. No openssl needed.

### 6. Install the Helm chart

```bash
helm install mtls charts/mtls-forward-proxy -f charts/mtls-forward-proxy/values-lab.yaml
```

`values-lab.yaml` maps `api.example.com` to the in-cluster demo webserver service. See the [chart README](charts/mtls-forward-proxy/README.md) for full configuration options.

### 7. Trust the wildcard cert in your browser (macOS)

```bash
# Add to login keychain only — NOT the system keychain (avoids breaking Docker TLS)
security add-trusted-cert -d -r trustRoot \
  -k ~/Library/Keychains/login.keychain-db \
  envoy-proxy/certs/wildcard.example.com.crt
```

Restart Chrome/Safari after adding.

### 8. Test

```bash
# Port-forward the proxy (keep this terminal open)
kubectl port-forward svc/mtls-envoy 3128:3128

# curl from your laptop
curl -x http://localhost:3128 https://api.example.com -k
```

Set your browser proxy to `localhost:3128` and navigate to `https://api.example.com`. The browser sends CONNECT — Envoy resolves and dials the backend over mTLS. No client DNS resolution required.

Expected response body:
```
mTLS handshake successful.
Client certificate: CN=scanner_sysid,O=lab
```

## Certificate relationships

```
lab-ca  (keypair generated in-cluster — secret: cert-manager/cert-manager-lab-ca-new)
  │  ClusterIssuer: lab-ca-issuer
  ├── *.example.com          — Envoy presents to browser (wildcard)
  └── CN=scanner_sysid       — Envoy presents to every backend (client cert)

webserver-ca  (keypair generated in-cluster — secret: cert-manager/cert-manager-webserver-ca-new)
  │  ClusterIssuer: webserver-ca-issuer
  ├── api.example.com        — nginx server cert
  ├── docs.example.com       — nginx server cert
  └── fred.example.com       — nginx server cert  (and any future backends)

envoy-upstream-ca  (cert only, no private key — secret: default/envoy-upstream-ca)
  └── webserver-ca.crt       — Envoy uses this to verify backend server certs
```

No private keys exist on disk or were ever generated outside the cluster.

## How it works

1. Browser sends `CONNECT api.example.com:443` to Envoy on port 3128
2. Envoy returns `200 OK` — CONNECT tunnel established
3. Browser performs TLS; Envoy terminates it with the `*.example.com` wildcard cert
4. Envoy's dynamic forward proxy resolves the hostname via cluster DNS
5. Envoy opens an mTLS connection to the backend, presenting `CN=scanner_sysid`
6. Backend verifies the client cert against the lab CA and responds
7. Response travels back through the tunnel to the browser

## Spec and design docs

| File | Description |
|------|-------------|
| [spec/envoy-mtls-offload-proxy-spec.md](spec/envoy-mtls-offload-proxy-spec.md) | Original requirements and design |
| [spec/envoy-config-explainer.md](spec/envoy-config-explainer.md) | Line-by-line Envoy config walkthrough |
| [spec/technology-selection.md](spec/technology-selection.md) | Envoy vs Squid vs HAProxy comparison |
| [spec/implementation-journal.md](spec/implementation-journal.md) | Lessons learned and iteration log |
| [spec/security-review.md](spec/security-review.md) | Security findings and recommendations |
