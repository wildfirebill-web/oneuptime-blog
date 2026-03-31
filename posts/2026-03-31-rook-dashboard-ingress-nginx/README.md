# How to Expose the Ceph Dashboard via Ingress with Nginx in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Dashboard, Ingress, Kubernetes

Description: Configure Nginx Ingress to expose the Rook-Ceph Dashboard with TLS termination and SSL passthrough for secure external access.

---

## Overview

Exposing the Ceph Dashboard through Nginx Ingress is the preferred method for clusters that already use an Ingress controller for other services. It allows centralized TLS management and hostname-based routing. Because the dashboard uses HTTPS internally, Nginx must be configured for SSL passthrough or TLS re-encryption.

## Prerequisites

- Nginx Ingress Controller installed in the cluster
- Rook-Ceph with dashboard enabled
- A DNS record pointing your desired hostname to the Ingress controller

## Option 1 - SSL Passthrough

SSL passthrough tunnels the TLS connection directly to the dashboard pod, preserving the Ceph-generated certificate end-to-end:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rook-ceph-dashboard-ingress
  namespace: rook-ceph
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
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

Enable SSL passthrough in the Nginx controller if not already enabled:

```bash
kubectl edit deployment ingress-nginx-controller -n ingress-nginx
# Add --enable-ssl-passthrough to the args
```

Or via Helm values:

```yaml
controller:
  extraArgs:
    enable-ssl-passthrough: ""
```

## Option 2 - TLS Re-Encryption

Terminate TLS at the Ingress and re-encrypt to the backend. This requires a TLS secret for the Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rook-ceph-dashboard-ingress
  namespace: rook-ceph
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "off"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - ceph-dashboard.example.com
    secretName: ceph-dashboard-tls
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

Create the TLS secret (or let cert-manager provision it):

```bash
kubectl create secret tls ceph-dashboard-tls \
  --cert=tls.crt --key=tls.key -n rook-ceph
```

## Verify the Ingress

```bash
kubectl get ingress -n rook-ceph
kubectl describe ingress rook-ceph-dashboard-ingress -n rook-ceph
```

Test access:

```bash
curl -k https://ceph-dashboard.example.com
```

## Retrieve Dashboard Credentials

```bash
kubectl -n rook-ceph get secret rook-ceph-dashboard-password \
  -o jsonpath="{['data']['password']}" | base64 --decode && echo
```

## Summary

Nginx Ingress provides flexible options for exposing the Ceph Dashboard - SSL passthrough for minimal configuration or TLS re-encryption for centralized certificate management with cert-manager. Both approaches integrate with standard hostname-based routing and make the dashboard accessible at a memorable URL without exposing NodePorts.
