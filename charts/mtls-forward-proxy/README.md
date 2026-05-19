# mtls-forward-proxy Helm Chart

Deploys the Envoy mTLS-offloading forward proxy and (optionally) the demo nginx backend into a Kubernetes cluster.

```
Browser ──CONNECT──► Envoy :3128 ──mTLS──► nginx :443
                      │ terminates downstream TLS
                      │ injects X-Forwarded-Client-Cert
```

## Prerequisites

- Kubernetes 1.24+
- Helm 3.x
- `openssl` (for cert generation)
- `kubectl` configured against your target cluster

---

## 1 — Generate certificates

All private keys stay on your machine and enter the cluster only as Kubernetes Secrets.

### 1a — Lab CA + client certificate (Envoy → upstream)

```bash
cd envoy-proxy/client-certs

# CA
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -subj "/CN=lab-ca" -out ca.crt

# Client cert signed by the lab CA
openssl genrsa -out client.key 2048
openssl req -new -key client.key \
  -subj "/CN=scanner_sysid/O=lab" -out client.csr
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -sha256 -days 3650 -out client.crt
```

### 1b — Per-hostname downstream certificates (Envoy → browser)

Run once for each hostname you want the proxy to intercept.

```bash
cd envoy-proxy/certs

for HOST in api.example.com app.example.com auth.example.com; do
  openssl req -x509 -nodes -newkey rsa:2048 -days 3650 \
    -subj "/CN=${HOST}" \
    -addext "subjectAltName=DNS:${HOST}" \
    -keyout "${HOST}.key" -out "${HOST}.crt"
done
```

> Add additional hostnames by extending the loop and adding a matching entry under `envoy.upstreams` in `values.yaml`.

### 1c — Web-server CA + server certificate (nginx)

```bash
cd web-server/certs

# Webserver CA
openssl genrsa -out webserver-ca.key 4096
openssl req -x509 -new -nodes -key webserver-ca.key -sha256 -days 3650 \
  -subj "/CN=webserver-ca" -out webserver-ca.crt

# nginx server cert signed by the webserver CA
openssl genrsa -out server.key 2048
openssl req -new -key server.key \
  -subj "/CN=api.example.com" -out server.csr
openssl x509 -req -in server.csr \
  -CA webserver-ca.crt -CAkey webserver-ca.key -CAcreateserial \
  -sha256 -days 3650 \
  -extfile <(printf "subjectAltName=DNS:api.example.com") \
  -out server.crt
```

---

## 2 — Create Kubernetes Secrets

The chart reads all certificates from pre-existing Secrets. Create them once; they persist across upgrades.

```bash
# Envoy downstream certs (browser-facing, one cert per proxied hostname)
kubectl create secret generic envoy-downstream-certs \
  --from-file=api.example.com.crt=envoy-proxy/certs/api.example.com.crt \
  --from-file=api.example.com.key=envoy-proxy/certs/api.example.com.key \
  --from-file=app.example.com.crt=envoy-proxy/certs/app.example.com.crt \
  --from-file=app.example.com.key=envoy-proxy/certs/app.example.com.key \
  --from-file=auth.example.com.crt=envoy-proxy/certs/auth.example.com.crt \
  --from-file=auth.example.com.key=envoy-proxy/certs/auth.example.com.key

# Envoy client certs (Envoy's identity when connecting upstream)
kubectl create secret generic envoy-client-certs \
  --from-file=client.crt=envoy-proxy/client-certs/client.crt \
  --from-file=client.key=envoy-proxy/client-certs/client.key \
  --from-file=ca.crt=envoy-proxy/client-certs/ca.crt

# CA that signed the nginx server cert (Envoy uses this to verify upstream)
kubectl create secret generic envoy-upstream-ca \
  --from-file=webserver-ca.crt=web-server/certs/webserver-ca.crt

# nginx certs (server cert/key + client CA for mTLS enforcement)
kubectl create secret generic nginx-certs \
  --from-file=server.crt=web-server/certs/server.crt \
  --from-file=server.key=web-server/certs/server.key \
  --from-file=client-ca.crt=envoy-proxy/client-certs/ca.crt
```

Verify all four secrets exist:

```bash
kubectl get secrets envoy-downstream-certs envoy-client-certs envoy-upstream-ca nginx-certs
```

---

## 3 — Install the chart

```bash
# From the repo root
helm install mtls charts/mtls-forward-proxy
```

The default release name is `mtls`. The Envoy pod connects to the demo webserver using the address in `values.yaml` (`envoy.upstreams[0].address`). If you use a different release name, the webserver Service name changes — update the value accordingly:

```bash
helm install my-release charts/mtls-forward-proxy \
  --set envoy.upstreams[0].address=my-release-webserver.default.svc.cluster.local
```

### Upgrade

```bash
helm upgrade mtls charts/mtls-forward-proxy
```

The Envoy Deployment has a `checksum/config` annotation so a `values.yaml` change automatically triggers a rolling restart.

### Uninstall

```bash
helm uninstall mtls
# Secrets are not managed by the chart — delete manually if needed:
kubectl delete secret envoy-downstream-certs envoy-client-certs envoy-upstream-ca nginx-certs
```

---

## 4 — Test the proxy

### 4a — One-shot curl pod

```bash
kubectl run curl-test \
  --image=curlimages/curl --rm -it --restart=Never -- \
  curl -x http://mtls-envoy.default.svc.cluster.local:3128 \
  https://api.example.com -k -v
```

Look for:
- HTTP/1.1 200 connection established (CONNECT phase)
- `mTLS handshake successful` in the response body
- `Client certificate: CN=scanner_sysid` in the response body

### 4b — Port-forward and test from your laptop

```bash
kubectl port-forward svc/mtls-envoy 3128:3128 &

curl -x http://localhost:3128 https://api.example.com -k
```

### 4c — Check Envoy admin stats

```bash
kubectl port-forward deployment/mtls-envoy 9901:9901 &
curl http://localhost:9901/stats | grep upstream_cx_total
```

---

## Configuration reference

| Key | Default | Description |
|-----|---------|-------------|
| `envoy.image.tag` | `v1.38-latest` | Envoy image tag |
| `envoy.replicaCount` | `1` | Number of Envoy pods |
| `envoy.service.type` | `ClusterIP` | Service type (`ClusterIP`, `NodePort`, `LoadBalancer`) |
| `envoy.service.proxyPort` | `3128` | Proxy listener port |
| `envoy.logLevel` | `info` | Envoy log level |
| `envoy.secrets.downstreamCerts` | `envoy-downstream-certs` | Secret with per-hostname downstream certs |
| `envoy.secrets.clientCerts` | `envoy-client-certs` | Secret with Envoy client cert and lab CA |
| `envoy.secrets.upstreamCA` | `envoy-upstream-ca` | Secret with upstream server CA |
| `envoy.upstreams` | (3 example entries) | List of proxied hostnames with upstream addresses |
| `webserver.enabled` | `true` | Deploy the demo nginx backend |
| `webserver.secret` | `nginx-certs` | Secret with nginx server cert/key and client CA |
| `namespace` | `default` | Namespace to deploy into |
