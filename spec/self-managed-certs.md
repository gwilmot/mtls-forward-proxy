# Self-Managed Certificates (OpenSSL)

Use this guide when cert-manager is not available. All certificates are generated locally with OpenSSL and loaded into Kubernetes secrets manually.

> **Security note:** Private keys will exist on disk during this process. Delete them once the secrets are loaded.

---

## Prerequisites

- `openssl` installed locally
- `kubectl` configured against your cluster

---

## 1. Create the CA keypairs

Two CAs are used:
- **lab-ca** — signs the Envoy wildcard downstream cert and the Envoy client cert
- **webserver-ca** — signs all backend server certs

```bash
# Lab CA
openssl genrsa -out lab-ca.key 4096
openssl req -new -x509 -days 3650 -key lab-ca.key -out lab-ca.crt -subj "/CN=lab-ca"

# Webserver CA
openssl genrsa -out webserver-ca.key 4096
openssl req -new -x509 -days 3650 -key webserver-ca.key -out webserver-ca.crt -subj "/CN=webserver-ca"
```

---

## 2. Wildcard downstream cert

Envoy presents this to browsers for all `*.example.com` hostnames.

```bash
openssl genrsa -out wildcard.key 2048
openssl req -new -key wildcard.key -out wildcard.csr -subj "/CN=*.example.com"
openssl x509 -req -days 3650 -in wildcard.csr -CA lab-ca.crt -CAkey lab-ca.key \
  -CAcreateserial -out wildcard.crt \
  -extfile <(printf "subjectAltName=DNS:*.example.com")
```

---

## 3. Envoy client cert

Envoy presents this to every backend during the mTLS handshake to prove its identity.

```bash
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr -subj "/CN=scanner_sysid/O=lab"
openssl x509 -req -days 3650 -in client.csr -CA lab-ca.crt -CAkey lab-ca.key \
  -CAcreateserial -out client.crt \
  -extfile <(printf "extendedKeyUsage=clientAuth")
```

---

## 4. Backend server cert

Repeat for each backend. The cert must include both the public hostname **and** the Kubernetes service FQDN as SANs — Envoy dials the service FQDN (via `hostOverrides`) and validates the cert against that name.

```bash
# Replace these values for each backend
HOSTNAME=api.example.com
SERVICE_FQDN=mtls-webserver.default.svc.cluster.local
CERT_NAME=api-server

openssl genrsa -out ${CERT_NAME}.key 2048
openssl req -new -key ${CERT_NAME}.key -out ${CERT_NAME}.csr -subj "/CN=${HOSTNAME}"
openssl x509 -req -days 3650 -in ${CERT_NAME}.csr -CA webserver-ca.crt -CAkey webserver-ca.key \
  -CAcreateserial -out ${CERT_NAME}.crt \
  -extfile <(printf "subjectAltName=DNS:${HOSTNAME},DNS:${SERVICE_FQDN}")
```

---

## 5. Load into Kubernetes secrets

The secret names and key names below match the chart defaults. Do not change them unless you also update `values.yaml`.

```bash
# Envoy wildcard downstream cert
kubectl create secret generic envoy-downstream-certs \
  --from-file=tls.crt=wildcard.crt \
  --from-file=tls.key=wildcard.key

# Envoy client cert (+ lab CA for completeness)
kubectl create secret generic envoy-client-certs \
  --from-file=tls.crt=client.crt \
  --from-file=tls.key=client.key \
  --from-file=ca.crt=lab-ca.crt

# Upstream CA trust anchor — public cert only, no private key
kubectl create secret generic envoy-upstream-ca \
  --from-file=tls.crt=webserver-ca.crt

# Backend server cert
# ca.crt must be the lab CA (not the webserver CA) — nginx uses it to verify
# Envoy's client cert, which is signed by the lab CA
kubectl create secret generic nginx-certs \
  --from-file=tls.crt=api-server.crt \
  --from-file=tls.key=api-server.key \
  --from-file=ca.crt=lab-ca.crt
```

---

## 6. Clean up private keys

```bash
rm lab-ca.key webserver-ca.key wildcard.key client.key api-server.key
rm *.csr *.srl
```

Keep `lab-ca.crt` and `webserver-ca.crt` — you will need them to issue additional backend certs and to trust the wildcard cert in your browser.

---

## 7. Trust the wildcard cert in your browser (macOS)

```bash
security add-trusted-cert -r trustRoot \
  -k ~/Library/Keychains/login.keychain-db \
  lab-ca.crt
```

Fully quit and relaunch Chrome/Safari after adding.

---

## Adding a new backend

```bash
HOSTNAME=docs.example.com
SERVICE_FQDN=docs-webserver.default.svc.cluster.local
CERT_NAME=docs-server

openssl genrsa -out ${CERT_NAME}.key 2048
openssl req -new -key ${CERT_NAME}.key -out ${CERT_NAME}.csr -subj "/CN=${HOSTNAME}"
openssl x509 -req -days 3650 -in ${CERT_NAME}.csr -CA webserver-ca.crt -CAkey webserver-ca.key \
  -CAcreateserial -out ${CERT_NAME}.crt \
  -extfile <(printf "subjectAltName=DNS:${HOSTNAME},DNS:${SERVICE_FQDN}")

kubectl create secret generic docs-nginx-certs \
  --from-file=tls.crt=${CERT_NAME}.crt \
  --from-file=tls.key=${CERT_NAME}.key \
  --from-file=ca.crt=lab-ca.crt

rm ${CERT_NAME}.key ${CERT_NAME}.csr
```

Then add the hostname to `hostOverrides` in `values-lab.yaml` and run `helm upgrade`.
