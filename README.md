# mTLS Forward Proxy — Lab

An Envoy-based explicit forward proxy that offloads mTLS client certificate authentication on behalf of clients that do not hold a client certificate themselves.

## Architecture

```
Browser / curl (no client cert)
    │
    │  HTTP CONNECT api.example.com:443   (plaintext, port 3128)
    ▼
┌─────────────────────────────────────────────┐
│               Envoy Proxy (K8s)             │
│                                             │
│  1. Accept CONNECT request                  │
│  2. Terminate downstream TLS               │
│     - present api.example.com cert         │
│  3. Originate upstream mTLS connection     │
│     - present client cert (CN=scanner_sysid)│
│     - verify nginx server cert             │
│  4. Forward decrypted bytes                │
└─────────────────────────────────────────────┘
    │
    │  mTLS (client cert verified by nginx)
    ▼
demo-webserver (nginx, K8s)
    - requires client cert signed by lab CA
    - serves Hello World page showing client CN
```

## Components

| Directory | Description |
|-----------|-------------|
| `envoy-proxy/` | Envoy forward proxy — listens on port 3128, does mTLS upstream |
| `web-server/` | Demo nginx server — enforces mTLS client cert auth on port 443 |
| `spec/` | Original configuration specification |

## Prerequisites

- Kubernetes cluster (tested with [kind](https://kind.sigs.k8s.io/) via Docker Desktop)
- `kubectl` configured against your cluster
- `openssl` for certificate generation
- `gh` CLI for repo setup (optional)

## Setup

### 1. Clone the repo

```bash
git clone <repo-url>
cd mtls-forward-proxy
```

### 2. Generate certificates

All certificates are excluded from the repo. Run the following commands to generate them before deploying.

#### 2a. Lab CA (signs the proxy client certificate)

```bash
mkdir -p envoy-proxy/client-certs

openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes \
  -keyout envoy-proxy/client-certs/ca.key \
  -out    envoy-proxy/client-certs/ca.crt \
  -subj   "/CN=lab-ca"
```

#### 2b. Client certificate (presented by Envoy to upstream origins)

```bash
openssl req -newkey rsa:4096 -nodes \
  -keyout envoy-proxy/client-certs/client.key \
  -out    envoy-proxy/client-certs/client.csr \
  -subj   "/CN=scanner_sysid"

openssl x509 -req -days 365 -sha256 \
  -in    envoy-proxy/client-certs/client.csr \
  -CA    envoy-proxy/client-certs/ca.crt \
  -CAkey envoy-proxy/client-certs/ca.key \
  -CAcreateserial \
  -out   envoy-proxy/client-certs/client.crt
```

#### 2c. Per-hostname downstream certificates (presented by Envoy to the browser)

```bash
mkdir -p envoy-proxy/certs

for HOST in api.example.com app.example.com auth.example.com; do
  openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes \
    -keyout "envoy-proxy/certs/${HOST}.key" \
    -out    "envoy-proxy/certs/${HOST}.crt" \
    -subj   "/CN=${HOST}" \
    -addext "subjectAltName=DNS:${HOST}"
done
```

#### 2d. Web server CA and server certificate

```bash
mkdir -p web-server/certs

# Web server CA
openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes \
  -keyout web-server/certs/webserver-ca.key \
  -out    web-server/certs/webserver-ca.crt \
  -subj   "/CN=webserver-ca"

# Server certificate for api.example.com, signed by webserver CA
openssl req -newkey rsa:4096 -nodes \
  -keyout web-server/certs/server.key \
  -out    web-server/certs/server.csr \
  -subj   "/CN=api.example.com"

openssl x509 -req -days 365 -sha256 \
  -in    web-server/certs/server.csr \
  -CA    web-server/certs/webserver-ca.crt \
  -CAkey web-server/certs/webserver-ca.key \
  -CAcreateserial \
  -extfile <(printf "subjectAltName=DNS:api.example.com") \
  -out   web-server/certs/server.crt
```

### 3. Create Kubernetes secrets

```bash
# Envoy: client cert + lab CA
kubectl create secret generic envoy-client-certs \
  --from-file=client.crt=envoy-proxy/client-certs/client.crt \
  --from-file=client.key=envoy-proxy/client-certs/client.key \
  --from-file=ca.crt=envoy-proxy/client-certs/ca.crt \
  --dry-run=client -o yaml > envoy-proxy/k8s/secret-client-certs.yaml
kubectl apply -f envoy-proxy/k8s/secret-client-certs.yaml

# Envoy: per-hostname downstream certs
kubectl create secret generic envoy-downstream-certs \
  --from-file=api.example.com.crt=envoy-proxy/certs/api.example.com.crt \
  --from-file=api.example.com.key=envoy-proxy/certs/api.example.com.key \
  --from-file=app.example.com.crt=envoy-proxy/certs/app.example.com.crt \
  --from-file=app.example.com.key=envoy-proxy/certs/app.example.com.key \
  --from-file=auth.example.com.crt=envoy-proxy/certs/auth.example.com.crt \
  --from-file=auth.example.com.key=envoy-proxy/certs/auth.example.com.key \
  --dry-run=client -o yaml > envoy-proxy/k8s/secret-downstream-certs.yaml
kubectl apply -f envoy-proxy/k8s/secret-downstream-certs.yaml

# Envoy: webserver CA (to verify nginx's server cert upstream)
kubectl create secret generic envoy-upstream-ca \
  --from-file=webserver-ca.crt=web-server/certs/webserver-ca.crt \
  --dry-run=client -o yaml > envoy-proxy/k8s/secret-upstream-ca.yaml
kubectl apply -f envoy-proxy/k8s/secret-upstream-ca.yaml

# nginx: server cert + client CA (to verify Envoy's client cert)
kubectl create secret generic nginx-certs \
  --from-file=server.crt=web-server/certs/server.crt \
  --from-file=server.key=web-server/certs/server.key \
  --from-file=client-ca.crt=envoy-proxy/client-certs/ca.crt \
  --dry-run=client -o yaml > web-server/k8s/secret-nginx-certs.yaml
kubectl apply -f web-server/k8s/secret-nginx-certs.yaml
```

### 4. Deploy

```bash
# Envoy proxy
kubectl apply -f envoy-proxy/k8s/configmap.yaml
kubectl apply -f envoy-proxy/k8s/deployment.yaml
kubectl apply -f envoy-proxy/k8s/service.yaml

# Demo web server
kubectl apply -f web-server/k8s/configmap.yaml
kubectl apply -f web-server/k8s/deployment.yaml
kubectl apply -f web-server/k8s/service.yaml

# Verify both are running
kubectl get pod -l app=envoy-mtls-proxy
kubectl get pod -l app=demo-webserver
```

### 5. Local DNS

Add the demo hostname to `/etc/hosts`:

```bash
echo "127.0.0.1  api.example.com" | sudo tee -a /etc/hosts
```

### 6. Trust the downstream certificate (browser)

So the browser trusts the cert Envoy presents, add it to your system trust store:

```bash
# macOS
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain \
  envoy-proxy/certs/api.example.com.crt
```

For Chrome on macOS, also add to the login keychain and fully restart Chrome:

```bash
sudo security add-trusted-cert -d -r trustRoot \
  -k ~/Library/Keychains/login.keychain-db \
  envoy-proxy/certs/api.example.com.crt
```

### 7. Access the demo

Start a port-forward in a terminal and leave it running:

```bash
kubectl port-forward svc/envoy-mtls-proxy 3128:3128 2>/dev/null
```

Configure your browser or curl to use `localhost:3128` as an HTTP proxy, then browse to `https://api.example.com`.

**curl:**
```bash
curl --proxy http://localhost:3128 --insecure https://api.example.com/
```

You should see:
```
Hello World
mTLS handshake successful.
Client certificate: CN=scanner_sysid
```

This confirms Envoy injected the client certificate into the upstream mTLS session with nginx, on behalf of a client that holds no certificate of its own.

## How it works

1. The browser sends `CONNECT api.example.com:443 HTTP/1.1` to Envoy on port 3128
2. Envoy returns `200 OK` — the CONNECT tunnel is established
3. The browser performs a TLS handshake; Envoy terminates it using `envoy-proxy/certs/api.example.com.crt`
4. Envoy opens a separate mTLS connection to nginx, presenting `client-certs/client.crt` (CN=scanner_sysid)
5. nginx verifies the client cert against the lab CA and serves the response
6. The decrypted HTTP response travels back through the tunnel to the browser

## Certificate relationships

```
lab-ca
  └── client.crt  (CN=scanner_sysid)  — Envoy presents to nginx
  
webserver-ca
  └── server.crt  (CN=api.example.com) — nginx presents to Envoy

api.example.com.crt (self-signed)      — Envoy presents to browser
```
