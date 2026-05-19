# mtls-forward-proxy Helm Chart

Deploys the Envoy mTLS-offloading forward proxy and (optionally) the demo nginx backend into a Kubernetes cluster.

```
Browser ──CONNECT──► Envoy :3128 ──mTLS──► any backend
                      │ terminates downstream TLS (wildcard cert)
                      │ resolves upstream via DNS (dynamic forward proxy)
                      │ presents client cert to every upstream
```

The proxy uses Envoy's **dynamic forward proxy** cluster — no static upstream list required. Add a new backend and it just works, as long as the hostname resolves via DNS and your wildcard downstream cert covers it.

## Prerequisites

- Kubernetes 1.24+
- Helm 3.x
- `openssl` (for cert generation)
- `kubectl` configured against your target cluster

---

## 1 — Generate certificates

All private keys stay on your machine and enter the cluster only as Kubernetes Secrets.

### 1a — Lab CA + client certificate (Envoy → upstream backends)

```bash
cd envoy-proxy/client-certs

# CA
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -subj "/CN=lab-ca" -out ca.crt

# Client cert signed by the lab CA (this is Envoy's identity to all backends)
openssl genrsa -out client.key 2048
openssl req -new -key client.key \
  -subj "/CN=scanner_sysid/O=lab" -out client.csr
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -sha256 -days 3650 -out client.crt
```

### 1b — Wildcard downstream certificate (Envoy → browser)

A single wildcard cert covers all proxied hostnames under your domain.

```bash
cd envoy-proxy/certs

openssl req -x509 -nodes -newkey rsa:2048 -days 3650 \
  -subj "/CN=*.example.com" \
  -addext "subjectAltName=DNS:*.example.com" \
  -keyout wildcard.example.com.key \
  -out wildcard.example.com.crt
```

> For a different domain, change the CN and SAN, then update `envoy.downstreamTLS.certFile` and `keyFile` in `values.yaml`.

### 1c — Web-server CA + server certificate (nginx)

The nginx server cert must include all SANs that Envoy will use to connect to it — both the client-facing hostname **and** the Kubernetes service FQDN.

```bash
cd web-server/certs

# Webserver CA
openssl genrsa -out webserver-ca.key 4096
openssl req -x509 -new -nodes -key webserver-ca.key -sha256 -days 3650 \
  -subj "/CN=webserver-ca" -out webserver-ca.crt

# nginx server cert — include both the public hostname and the K8s service FQDN
openssl genrsa -out server.key 2048
openssl req -new -key server.key \
  -subj "/CN=api.example.com" -out server.csr
openssl x509 -req -in server.csr \
  -CA webserver-ca.crt -CAkey webserver-ca.key -CAcreateserial \
  -sha256 -days 3650 \
  -extfile <(printf "subjectAltName=DNS:api.example.com,DNS:<release>-webserver.<namespace>.svc.cluster.local") \
  -out server.crt
```

> Replace `<release>` and `<namespace>` with your Helm release name and namespace (e.g. `mtls` and `default`).
> This is necessary because `hostOverrides` rewrites the upstream Host header to the K8s service FQDN, which Envoy then validates against the server cert's SAN list.

---

## 2 — Create Kubernetes Secrets

The chart reads all certificates from pre-existing Secrets. Create them once; they persist across upgrades.

```bash
# Envoy downstream wildcard cert (browser-facing)
kubectl create secret generic envoy-downstream-certs \
  --from-file=wildcard.example.com.crt=envoy-proxy/certs/wildcard.example.com.crt \
  --from-file=wildcard.example.com.key=envoy-proxy/certs/wildcard.example.com.key

# Envoy client certs (Envoy's identity when connecting upstream)
kubectl create secret generic envoy-client-certs \
  --from-file=client.crt=envoy-proxy/client-certs/client.crt \
  --from-file=client.key=envoy-proxy/client-certs/client.key \
  --from-file=ca.crt=envoy-proxy/client-certs/ca.crt

# CA that signed all backend server certs (Envoy uses this to verify upstreams)
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
helm install mtls charts/mtls-forward-proxy -f charts/mtls-forward-proxy/values-lab.yaml
```

`values-lab.yaml` sets `hostOverrides` to map `api.example.com` to the Helm-managed webserver service. For a real deployment where DNS resolves your backends directly, no override file is needed:

```bash
helm install mtls charts/mtls-forward-proxy
```

### Upgrade

```bash
helm upgrade mtls charts/mtls-forward-proxy -f charts/mtls-forward-proxy/values-lab.yaml
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
  https://api.example.com -k -s
```

Expected response:
```html
<p>mTLS handshake successful.</p>
<p>Client certificate: CN=scanner_sysid</p>
```

### 4b — Port-forward and test from your laptop

```bash
kubectl port-forward svc/mtls-envoy 3128:3128 &
curl -x http://localhost:3128 https://api.example.com -k
```

### 4c — Check Envoy admin stats

```bash
kubectl port-forward deployment/mtls-envoy 9901:9901 &
curl -s http://localhost:9901/stats | grep upstream_cx_total
```

---

## Adding backends

The proxy requires no static upstream configuration. Any hostname that:
1. Is covered by the wildcard downstream cert (e.g. `*.example.com`)
2. Resolves via DNS to an mTLS-capable backend
3. Has a server cert signed by the upstream CA in `envoy-upstream-ca`

...will work automatically. For in-cluster services where the client-facing hostname doesn't match the K8s service DNS name, add an entry to `hostOverrides`:

```yaml
envoy:
  hostOverrides:
    - hostname: api.example.com
      address: api-deployment.default.svc.cluster.local
    - hostname: payments.example.com
      address: payments-svc.payments.svc.cluster.local
```

The server cert for each backend must include both the public hostname and the K8s service FQDN as SANs.

---

## Configuration reference

| Key | Default | Description |
|-----|---------|-------------|
| `envoy.image.tag` | `v1.38-latest` | Envoy image tag |
| `envoy.replicaCount` | `1` | Number of Envoy pods |
| `envoy.service.type` | `ClusterIP` | Service type (`ClusterIP`, `NodePort`, `LoadBalancer`) |
| `envoy.service.proxyPort` | `3128` | Proxy listener port |
| `envoy.logLevel` | `info` | Envoy log level |
| `envoy.downstreamTLS.certFile` | `wildcard.example.com.crt` | Wildcard cert filename in the downstreamCerts secret |
| `envoy.downstreamTLS.keyFile` | `wildcard.example.com.key` | Wildcard key filename in the downstreamCerts secret |
| `envoy.secrets.downstreamCerts` | `envoy-downstream-certs` | Secret with the wildcard downstream cert |
| `envoy.secrets.clientCerts` | `envoy-client-certs` | Secret with Envoy client cert and lab CA |
| `envoy.secrets.upstreamCA` | `envoy-upstream-ca` | Secret with upstream server CA |
| `envoy.hostOverrides` | `[]` | Hostname → backend address mappings for in-cluster services |
| `webserver.enabled` | `true` | Deploy the demo nginx backend |
| `webserver.secret` | `nginx-certs` | Secret with nginx server cert/key and client CA |
| `namespace` | `default` | Namespace to deploy into |
