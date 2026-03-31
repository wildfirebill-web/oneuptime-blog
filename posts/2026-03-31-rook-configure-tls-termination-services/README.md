# How to Configure TLS Termination for Rook-Ceph Services

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, TLS, SSL, Security, Kubernetes, Certificate

Description: Configure TLS encryption for Rook-Ceph services including RGW, dashboard, and monitors using cert-manager or custom certificates for secure communication.

---

## Overview

TLS termination for Rook-Ceph services can be handled at multiple layers: within Ceph services themselves, at an Ingress controller, or via a service mesh. This guide covers configuring TLS for RGW and the dashboard using both native Ceph TLS and Ingress-based termination.

## TLS for the Ceph Dashboard

Enable native TLS on the dashboard:

```yaml
spec:
  dashboard:
    enabled: true
    port: 8443
    ssl: true
```

Rook creates a self-signed certificate automatically. To use a custom certificate:

```bash
kubectl -n rook-ceph create secret tls rook-ceph-mgr-dashboard-tls \
  --cert=dashboard.crt \
  --key=dashboard.key
```

Reference in CephCluster:

```yaml
spec:
  dashboard:
    ssl: true
    urlPrefix: /
```

## TLS for RGW (S3 API)

Configure HTTPS on the RGW:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  gateway:
    port: 80
    securePort: 443
    sslCertificateRef: rgw-tls-cert
    instances: 2
```

Create the certificate secret referenced by `sslCertificateRef`:

```bash
kubectl -n rook-ceph create secret tls rgw-tls-cert \
  --cert=rgw.crt \
  --key=rgw.key
```

Or use cert-manager:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: rgw-tls-cert
  namespace: rook-ceph
spec:
  secretName: rgw-tls-cert
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - s3.example.com
```

## TLS at the Ingress Layer

For Ingress-based TLS termination (Ceph services run HTTP internally):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rgw-tls
  namespace: rook-ceph
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - s3.example.com
    secretName: rgw-ingress-tls
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

## TLS for Ceph Monitors

For encrypted monitor connections (msgr2 protocol), enable:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mon ms_cluster_mode secure

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd ms_cluster_mode secure
```

Verify msgr2 encryption status:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mon dump | grep secure
```

## Verifying TLS Configuration

Test RGW HTTPS:

```bash
curl -v https://s3.example.com/
openssl s_client -connect s3.example.com:443 -showcerts
```

Test dashboard TLS:

```bash
curl -vk https://ceph.example.com:8443/
```

## Summary

TLS for Rook-Ceph services is configured either natively in Ceph using the `sslCertificateRef` for RGW and `ssl: true` for the dashboard, or at the Ingress layer for centralized certificate management. Use cert-manager to automate certificate provisioning and renewal, and enable msgr2 secure mode for encrypted cluster-internal communication.
