# How to Set Up SSL Certificates for Rook Object Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Object Store, SSL, TLS, S3, Kubernetes, Security

Description: Configure SSL/TLS certificates for the Rook-Ceph object store gateway so that S3-compatible clients connect over HTTPS with trusted certificate validation.

---

## Why Enable SSL for the Object Store

By default, the Rook CephObjectStore gateway (RGW) listens on HTTP. Any S3 client communicating over HTTP sends credentials and data in plaintext. For production deployments, TLS must be enabled so that all traffic between clients and the gateway is encrypted. Rook supports both externally issued certificates stored in Kubernetes Secrets and certificate-manager-issued certificates.

## Creating a TLS Secret for RGW

Prepare a TLS certificate and private key, then store them in a Kubernetes Secret in the `rook-ceph` namespace. The Secret must use the `kubernetes.io/tls` type:

```bash
# Generate a self-signed certificate for testing
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=rook-ceph-rgw-my-store.rook-ceph.svc" \
  -addext "subjectAltName=DNS:rook-ceph-rgw-my-store.rook-ceph.svc,DNS:s3.example.com"

kubectl -n rook-ceph create secret tls rook-ceph-rgw-tls-cert \
  --cert=tls.crt \
  --key=tls.key
```

For production, replace the self-signed certificate with one issued by your CA or cert-manager.

## Configuring SSL in the CephObjectStore CRD

Reference the TLS Secret in the CephObjectStore spec under `gateway.sslCertificateRef`:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  metadataPool:
    failureDomain: host
    replicated:
      size: 3
  dataPool:
    failureDomain: host
    replicated:
      size: 3
  preservePoolsOnDelete: true
  gateway:
    port: 80
    securePort: 443
    instances: 1
    sslCertificateRef: rook-ceph-rgw-tls-cert
```

Setting `securePort: 443` and `sslCertificateRef` enables HTTPS. You can keep `port: 80` for HTTP or set it to `0` to disable unencrypted access.

## Disabling HTTP After Enabling HTTPS

To force all clients to use HTTPS, remove the `port` field or set it to `0`:

```yaml
  gateway:
    port: 0
    securePort: 443
    instances: 1
    sslCertificateRef: rook-ceph-rgw-tls-cert
```

Apply the updated manifest and verify that the RGW pod restarts with the new configuration:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-rgw
kubectl -n rook-ceph logs -l app=rook-ceph-rgw | grep -i ssl
```

## Using cert-manager for Automatic Certificate Rotation

If cert-manager is installed in the cluster, create a Certificate resource and reference its generated Secret:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: rook-rgw-cert
  namespace: rook-ceph
spec:
  secretName: rook-ceph-rgw-tls-cert
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - s3.example.com
    - rook-ceph-rgw-my-store.rook-ceph.svc
```

cert-manager renews the certificate automatically before expiry. The Rook operator watches the Secret and restarts RGW pods when the certificate content changes.

## Testing the HTTPS Endpoint

Use the AWS CLI to verify the HTTPS endpoint works:

```bash
# Get the RGW service endpoint
kubectl -n rook-ceph get svc rook-ceph-rgw-my-store

# Test with AWS CLI (use --no-verify-ssl for self-signed)
AWS_ACCESS_KEY_ID=<key> AWS_SECRET_ACCESS_KEY=<secret> \
  aws s3 ls \
  --endpoint-url https://rook-ceph-rgw-my-store.rook-ceph.svc:443 \
  --no-verify-ssl
```

For a CA-signed certificate, omit `--no-verify-ssl`.

## Exposing RGW via Ingress with TLS Termination

An alternative approach terminates TLS at the Ingress or load balancer and leaves RGW listening on plain HTTP internally:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rook-rgw-ingress
  namespace: rook-ceph
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
spec:
  tls:
    - hosts:
        - s3.example.com
      secretName: ingress-tls-cert
  rules:
    - host: s3.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rook-ceph-rgw-my-store
                port:
                  number: 80
```

This approach is simpler for clusters already using a centralized Ingress controller with TLS management.

## Summary

SSL for the Rook CephObjectStore is enabled by creating a `kubernetes.io/tls` Secret containing the certificate and key, then referencing it via `gateway.sslCertificateRef` in the CephObjectStore spec. Use cert-manager for automatic renewal in production, disable plain HTTP by setting `gateway.port: 0`, and verify connectivity with the AWS CLI after deployment.
