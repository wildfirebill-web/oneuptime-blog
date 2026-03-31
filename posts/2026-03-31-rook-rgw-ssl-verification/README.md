# How to Configure SSL Verification for Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, SSL, TLS, Security

Description: Configure SSL/TLS certificate verification settings in Ceph RGW for secure HTTPS communication with S3 clients and backend connections.

---

Enabling SSL in Ceph RGW ensures that data in transit between clients and the object gateway is encrypted. This guide covers configuring SSL for the RGW frontend and tuning certificate verification behavior.

## Configuring SSL in the Beast Frontend

The beast frontend supports native TLS via OpenSSL:

```bash
# Set the frontend with SSL
ceph config set client.rgw rgw_frontends \
  "beast ssl_port=443 ssl_certificate=/etc/ceph/rgw.crt ssl_private_key=/etc/ceph/rgw.key"
```

Both HTTP and HTTPS simultaneously:

```bash
ceph config set client.rgw rgw_frontends \
  "beast port=80 ssl_port=443 ssl_certificate=/etc/ceph/rgw.crt ssl_private_key=/etc/ceph/rgw.key"
```

## Setting SSL Verification for Backend Connections

When RGW makes outbound HTTPS calls (e.g., to Keystone or external endpoints), control certificate verification:

```bash
# Disable SSL verification for self-signed certs (dev/test only)
ceph config set client.rgw rgw_verify_ssl false

# Enable SSL verification (production default)
ceph config set client.rgw rgw_verify_ssl true
```

## Generating a Self-Signed Certificate for Testing

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ceph/rgw.key \
  -out /etc/ceph/rgw.crt \
  -subj "/CN=rook-ceph-rgw-my-store.rook-ceph.svc"
```

## Injecting Certificates in Rook

Store the certificate and key as a Kubernetes Secret:

```bash
kubectl create secret tls rgw-tls-secret \
  --cert=/etc/ceph/rgw.crt \
  --key=/etc/ceph/rgw.key \
  -n rook-ceph
```

Reference in the CephObjectStore:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  gateway:
    instances: 1
    port: 80
    securePort: 443
    sslCertificateRef: rgw-tls-secret
```

## Verifying SSL Configuration

```bash
# Test HTTPS connectivity
curl -k https://rook-ceph-rgw-my-store.rook-ceph.svc/

# Check certificate details
openssl s_client -connect rook-ceph-rgw-my-store.rook-ceph.svc:443 < /dev/null 2>&1 | \
  openssl x509 -noout -dates -subject
```

## Client-Side SSL Configuration

Configure the AWS CLI to use a custom CA bundle:

```bash
aws s3 ls s3://my-bucket \
  --endpoint-url https://rook-ceph-rgw-my-store.rook-ceph.svc \
  --ca-bundle /etc/ceph/rgw.crt
```

## Summary

SSL in Ceph RGW is configured via `ssl_port`, `ssl_certificate`, and `ssl_private_key` in the `rgw_frontends` parameter. In Rook, use the `securePort` and `sslCertificateRef` fields in `CephObjectStore`. Set `rgw_verify_ssl = false` only in test environments with self-signed certificates - always use verified certificates in production.
