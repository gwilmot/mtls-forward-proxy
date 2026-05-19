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
| `charts/mtls-forward-proxy/` | Helm chart — primary way to deploy everything |
| `envoy-proxy/k8s/` | Raw kubectl manifests (reference / legacy) |
| `web-server/k8s/` | Raw kubectl manifests for the demo nginx backend |
| `spec/` | Architecture spec, config explainer, security review, tech selection |

## Prerequisites

- Kubernetes 1.24+ (tested with [kind](https://kind.sigs.k8s.io/) via Docker Desktop)
- Helm 3.x
- kubectl configured against your cluster
- cert-manager v1.17+ installed in the cluster
- `openssl` — only needed once to bootstrap the two CA keypairs

## Quick start

### 1. Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.2/cert-manager.yaml
kubectl rollout status deployment/cert-manager deployment/cert-manager-cainjector deployment/cert-manager-webhook \
  -n cert-manager --timeout=120s
```

### 2. Bootstrap the two CA keypairs (one-time, openssl)

These are the only keys that ever touch disk. After this step all leaf certs are issued by cert-manager in-cluster.

```bash
mkdir -p envoy-proxy/client-certs web-server/certs

# Lab CA — signs Envoy's client cert and the wildcard downstream cert
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes \
  -subj "/CN=lab-ca" \
  -keyout envoy-proxy/client-certs/ca.key \
  -out    envoy-proxy/client-certs/ca.crt

# Webserver CA — signs all backend server certs
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes \
  -subj "/CN=webserver-ca" \
  -keyout web-server/certs/webserver-ca.key \
  -out    web-server/certs/webserver-ca.crt
```

### 3. Load the CA keypairs into cert-manager

```bash
# ClusterIssuer secrets must live in the cert-manager namespace
kubectl create secret tls cert-manager-lab-ca \
  --cert=envoy-proxy/client-certs/ca.crt \
  --key=envoy-proxy/client-certs/ca.key \
  -n cert-manager

kubectl create secret tls cert-manager-webserver-ca \
  --cert=web-server/certs/webserver-ca.crt \
  --key=web-server/certs/webserver-ca.key \
  -n cert-manager

# Upstream CA trust anchor for Envoy (CA cert only, no key needed at runtime)
kubectl create secret generic envoy-upstream-ca \
  --from-file=tls.crt=web-server/certs/webserver-ca.crt
```

### 4. Create the ClusterIssuers

```bash
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: lab-ca-issuer
spec:
  ca:
    secretName: cert-manager-lab-ca
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: webserver-ca-issuer
spec:
  ca:
    secretName: cert-manager-webserver-ca
EOF

kubectl get clusterissuer   # both should show READY=True
```

### 5. Issue certificates via cert-manager

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
lab-ca  (cert-manager ClusterIssuer: lab-ca-issuer)
  ├── *.example.com          — Envoy presents to browser (wildcard)
  └── CN=scanner_sysid       — Envoy presents to every backend (client cert)

webserver-ca  (cert-manager ClusterIssuer: webserver-ca-issuer)
  ├── api.example.com        — nginx server cert
  ├── docs.example.com       — nginx server cert
  └── fred.example.com       — nginx server cert  (and any future backends)

envoy-upstream-ca  (static secret — CA cert only, no key)
  └── webserver-ca.crt       — Envoy uses this to verify backend server certs
```

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
