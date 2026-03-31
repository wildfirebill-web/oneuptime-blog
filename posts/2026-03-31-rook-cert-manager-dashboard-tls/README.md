# How to Set Up cert-manager for Ceph Dashboard TLS in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Certificate, Kubernetes, Security

Description: Configure cert-manager to issue and automatically renew TLS certificates for the Rook-Ceph Dashboard Ingress endpoint.

---

## Overview

By default, the Ceph Dashboard uses a self-signed certificate which triggers browser security warnings. Integrating cert-manager automates TLS certificate issuance from Let's Encrypt or an internal CA, providing trusted certificates that renew automatically before expiration.

## Prerequisites

- cert-manager installed in the cluster
- An Ingress controller (Nginx recommended)
- DNS record pointing your dashboard hostname to the Ingress IP

Verify cert-manager is running:

```bash
kubectl get pods -n cert-manager
kubectl get crd | grep cert-manager
```

## Create a ClusterIssuer

For Let's Encrypt production certificates:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

For an internal CA (useful for air-gapped environments):

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca
spec:
  ca:
    secretName: root-ca-secret
```

## Annotate the Ingress for cert-manager

Create or update the dashboard Ingress with cert-manager annotations:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rook-ceph-dashboard-ingress
  namespace: rook-ceph
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "off"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - ceph-dashboard.example.com
    secretName: ceph-dashboard-tls-cert
  rules:
  - host: ceph-dashboard.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: rook-ceph-mgr-dashboard
            port:
              number: 8443
```

## Monitor Certificate Issuance

cert-manager will automatically create a `Certificate` resource and begin the ACME challenge:

```bash
kubectl get certificate -n rook-ceph
kubectl describe certificate ceph-dashboard-tls-cert -n rook-ceph
kubectl get certificaterequest -n rook-ceph
kubectl get order -n rook-ceph
```

Once the certificate is issued, the `READY` column will show `True`:

```bash
kubectl get certificate ceph-dashboard-tls-cert -n rook-ceph
# NAME                       READY   SECRET                    AGE
# ceph-dashboard-tls-cert    True    ceph-dashboard-tls-cert   2m
```

## Verify Automatic Renewal

cert-manager renews certificates 30 days before expiration by default. Check renewal status:

```bash
kubectl describe certificate ceph-dashboard-tls-cert -n rook-ceph | grep -A5 "Renewal Time"
```

## Test the Certificate

```bash
curl -v https://ceph-dashboard.example.com 2>&1 | grep -E "subject|issuer|expire"
openssl s_client -connect ceph-dashboard.example.com:443 -showcerts 2>/dev/null | \
  openssl x509 -noout -text | grep -E "Subject:|Issuer:|Not After"
```

## Summary

Integrating cert-manager with the Rook-Ceph Dashboard Ingress removes self-signed certificate warnings and automates the entire certificate lifecycle. By annotating the Ingress resource with the appropriate `cert-manager.io/cluster-issuer` annotation, cert-manager handles issuance and renewal transparently with no manual certificate management required.
