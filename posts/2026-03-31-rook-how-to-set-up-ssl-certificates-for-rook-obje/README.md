# How to Set Up SSL Certificates for Rook Object Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Object Storage, Ssl, Tls, Security, Kubernetes

Description: Learn how to configure SSL/TLS certificates for the Rook Ceph Object Store to secure S3-compatible API traffic with HTTPS.

---

## Overview

Securing the Rook Object Store with SSL/TLS ensures that all S3 API traffic is encrypted. Rook supports configuring SSL certificates through Kubernetes Secrets, allowing you to use certificates from Let's Encrypt, cert-manager, or your own CA.

## Option 1 - Using cert-manager to Issue Certificates

If you have cert-manager installed, create a Certificate resource:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: rook-ceph-rgw-cert
  namespace: rook-ceph
spec:
  secretName: rook-ceph-rgw-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - s3.example.com
    - rook-ceph-rgw-my-store.rook-ceph.svc
```

## Option 2 - Creating a Self-Signed Certificate

Generate a self-signed certificate for testing:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=s3.example.com" \
  -addext "subjectAltName=DNS:s3.example.com,DNS:rook-ceph-rgw-my-store.rook-ceph.svc"
```

Create the Kubernetes Secret:

```bash
kubectl -n rook-ceph create secret tls rook-ceph-rgw-tls \
  --cert=tls.crt \
  --key=tls.key
```

## Option 3 - Using an Existing Certificate Secret

If you already have a certificate in a Secret:

```bash
kubectl -n rook-ceph create secret generic rook-ceph-rgw-tls \
  --from-file=tls.crt=./certificate.crt \
  --from-file=tls.key=./private.key \
  --from-file=ca.crt=./ca.crt
```

## Configuring the CephObjectStore for SSL

Reference the certificate Secret in the CephObjectStore spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPool:
    erasureCoded:
      dataChunks: 2
      codingChunks: 1
  preservePoolsOnDelete: true
  gateway:
    sslCertificateRef: rook-ceph-rgw-tls
    securePort: 443
    port: 80
    instances: 2
```

Apply the updated configuration:

```bash
kubectl apply -f object-store.yaml
```

## Verifying SSL Configuration

Check that the RGW pods restarted with the new certificate:

```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-rgw
```

Test the HTTPS endpoint:

```bash
curl -v https://s3.example.com/ \
  --cacert ca.crt \
  -H "Authorization: AWS4-HMAC-SHA256 ..."
```

For a quick connectivity check with a self-signed cert:

```bash
curl -v --insecure https://<node-ip>:443/
```

## Configuring S3 Clients for SSL

Update AWS CLI configuration to use the HTTPS endpoint:

```bash
aws configure set aws_access_key_id $ACCESS_KEY
aws configure set aws_secret_access_key $SECRET_KEY

# With a custom CA
aws --endpoint-url https://s3.example.com \
  --ca-bundle ca.crt \
  s3 ls
```

## Rotating Certificates

Update the Secret with the new certificate and key, then restart RGW pods:

```bash
kubectl -n rook-ceph create secret tls rook-ceph-rgw-tls \
  --cert=new-tls.crt \
  --key=new-tls.key \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl -n rook-ceph rollout restart deployment rook-ceph-rgw-my-store-a
```

## Summary

Setting up SSL for the Rook Object Store involves creating a Kubernetes Secret containing the TLS certificate and private key, then referencing that Secret via `sslCertificateRef` in the CephObjectStore spec. You can use cert-manager, self-signed certificates, or existing certificates from your PKI. Once configured, RGW serves HTTPS traffic on the `securePort`, encrypting all S3 API communications.
