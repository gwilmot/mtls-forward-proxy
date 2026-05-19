# mtls-forward-proxy Helm Chart

Deploys the Envoy mTLS-offloading forward proxy and an optional demo nginx backend.

```
Browser ──CONNECT──► Envoy :3128 ──mTLS──► any *.example.com backend
                      │ wildcard TLS termination (*.example.com)
                      │ dynamic forward proxy — DNS resolution at request time
                      │ client cert (CN=scanner_sysid) presented to every backend
```

Adding a new backend requires only a cert-manager `Certificate` resource — no changes to the proxy config or chart values.

## Prerequisites

- Kubernetes 1.24+
- Helm 3.x
- cert-manager v1.17+ with `selfsigned-issuer`, `lab-ca-issuer`, and `webserver-ca-issuer` ClusterIssuers configured
- `envoy-upstream-ca` secret containing the webserver CA cert (public cert only)
- No `openssl` required — all CA and leaf keypairs are generated in-cluster by cert-manager

See the [root README](../../README.md) for the full bootstrap procedure.

---

## Secrets

The chart reads four pre-existing secrets. All are issued by cert-manager except `envoy-upstream-ca`.

| Secret | Contents | Managed by |
|--------|----------|------------|
| `envoy-downstream-certs` | Wildcard `*.example.com` cert/key | cert-manager (`envoy-downstream-wildcard` Certificate) |
| `envoy-client-certs` | Envoy client cert `CN=scanner_sysid` + lab CA | cert-manager (`envoy-client-cert` Certificate) |
| `envoy-upstream-ca` | `webserver-ca.crt` trust anchor | Manual (CA cert only — nothing to rotate) |
| `nginx-certs` | Backend server cert/key + lab CA | cert-manager (`api-webserver-cert` Certificate) |

Secret key names follow the cert-manager defaults: `tls.crt`, `tls.key`, `ca.crt`.

---

## Install

```bash
# Lab install — maps api.example.com to the in-cluster demo webserver
helm install mtls charts/mtls-forward-proxy -f charts/mtls-forward-proxy/values-lab.yaml

# Production install — backends resolve via external DNS, no overrides needed
helm install mtls charts/mtls-forward-proxy
```

### Upgrade

```bash
helm upgrade mtls charts/mtls-forward-proxy -f charts/mtls-forward-proxy/values-lab.yaml
```

The Envoy Deployment has a `checksum/config` annotation — any change to values that affects the ConfigMap automatically triggers a rolling restart.

### Uninstall

```bash
helm uninstall mtls
# Secrets and cert-manager Certificate resources are not chart-managed — clean up manually:
kubectl delete certificate envoy-downstream-wildcard envoy-client-cert api-webserver-cert
kubectl delete secret envoy-downstream-certs envoy-client-certs envoy-upstream-ca nginx-certs
```

---

## Adding a backend

1. **Create a cert-manager Certificate** — private key stays in-cluster, never on disk:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: payments-webserver-cert
  namespace: default
spec:
  secretName: payments-nginx-certs
  commonName: payments.example.com
  dnsNames:
    - payments.example.com
    - payments-svc.default.svc.cluster.local   # K8s service FQDN
  issuerRef:
    name: webserver-ca-issuer
    kind: ClusterIssuer
  duration: 87600h
  renewBefore: 720h
```

2. **Deploy the nginx backend** mounting the cert-manager secret (`tls.crt`, `tls.key`, `ca.crt`).

3. **Add a `hostOverride`** if the client-facing hostname differs from the K8s service name (always true for in-cluster services):

```yaml
# values-lab.yaml
envoy:
  hostOverrides:
    - hostname: api.example.com
      address: mtls-webserver.default.svc.cluster.local
    - hostname: payments.example.com
      address: payments-svc.default.svc.cluster.local
```

```bash
helm upgrade mtls charts/mtls-forward-proxy -f charts/mtls-forward-proxy/values-lab.yaml
```

The wildcard downstream cert already covers the new hostname — no Envoy config changes needed.

---

## Testing

### From inside the cluster

```bash
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- \
  curl -x http://mtls-envoy.default.svc.cluster.local:3128 \
  https://api.example.com -k -s
```

### From your laptop

```bash
kubectl port-forward svc/mtls-envoy 3128:3128 &
curl -x http://localhost:3128 https://api.example.com -k
```

Expected response:
```
mTLS handshake successful.
Client certificate: CN=scanner_sysid,O=lab
```

### Envoy admin stats

```bash
kubectl port-forward deployment/mtls-envoy 9901:9901 &
curl -s http://localhost:9901/stats | grep dns_cache
curl -s http://localhost:9901/clusters | grep dynamic_forward
```

---

## hostOverrides explained

The dynamic forward proxy resolves the upstream hostname from the CONNECT request via DNS. For in-cluster services the client-facing hostname (`api.example.com`) won't resolve to the K8s service IP — you need an override so Envoy dials the correct address.

The override uses the DFP `PerRouteConfig` `host_rewrite_literal` mechanism, which runs inside the DFP filter before the DNS lookup. The backend server cert must include both the public hostname **and** the K8s service FQDN as SANs, because Envoy validates the cert against the rewritten (service FQDN) hostname.

For backends reachable via real external DNS (e.g. a public SaaS API), no override is needed.

---

## Configuration reference

| Key | Default | Description |
|-----|---------|-------------|
| `envoy.image.tag` | `v1.38-latest` | Envoy image tag |
| `envoy.replicaCount` | `1` | Number of Envoy pods |
| `envoy.service.type` | `ClusterIP` | `ClusterIP`, `NodePort`, or `LoadBalancer` |
| `envoy.service.proxyPort` | `3128` | Proxy listener port |
| `envoy.service.adminPort` | `9901` | Envoy admin port (localhost only inside pod) |
| `envoy.logLevel` | `info` | Envoy log level |
| `envoy.downstreamTLS.certFile` | `tls.crt` | Cert key name in `downstreamCerts` secret |
| `envoy.downstreamTLS.keyFile` | `tls.key` | Key name in `downstreamCerts` secret |
| `envoy.secrets.downstreamCerts` | `envoy-downstream-certs` | Secret with wildcard downstream cert |
| `envoy.secrets.clientCerts` | `envoy-client-certs` | Secret with Envoy client cert |
| `envoy.secrets.upstreamCA` | `envoy-upstream-ca` | Secret with upstream server CA cert |
| `envoy.hostOverrides` | `[]` | Hostname → K8s service address mappings |
| `webserver.enabled` | `true` | Deploy the demo nginx backend |
| `webserver.secret` | `nginx-certs` | Secret with nginx server cert/key and client CA |
| `namespace` | `default` | Namespace to deploy into |
