# How to Expose the Ceph Dashboard via LoadBalancer in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Dashboard, Kubernetes, Networking

Description: Learn how to expose the Ceph Dashboard using a Kubernetes LoadBalancer service in Rook for cloud or MetalLB-enabled bare-metal clusters.

---

## Overview

Using a LoadBalancer service to expose the Ceph Dashboard is the preferred approach for cloud-based Kubernetes clusters (GKE, EKS, AKS) and bare-metal clusters using MetalLB. It assigns a stable external IP directly to the dashboard service, avoiding NodePort limitations.

## Prerequisites

- A Kubernetes cluster with LoadBalancer support (cloud provider or MetalLB configured)
- Rook-Ceph deployed with the dashboard enabled

Verify the dashboard is running:

```bash
kubectl get pods -n rook-ceph -l app=rook-ceph-mgr
kubectl get svc -n rook-ceph rook-ceph-mgr-dashboard
```

## Create a LoadBalancer Service

Create a new service of type `LoadBalancer` pointing to the Ceph MGR dashboard:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rook-ceph-dashboard-lb
  namespace: rook-ceph
  annotations:
    # For AWS ELB
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    # For GCP - add any needed annotations here
spec:
  type: LoadBalancer
  selector:
    app: rook-ceph-mgr
    rook_cluster: rook-ceph
  ports:
  - name: https-dashboard
    port: 443
    targetPort: 8443
    protocol: TCP
```

```bash
kubectl apply -f dashboard-lb.yaml
kubectl get svc -n rook-ceph rook-ceph-dashboard-lb
```

Wait for the `EXTERNAL-IP` to be assigned, then access `https://<external-ip>`.

## MetalLB Configuration Example

For bare-metal clusters using MetalLB, assign a specific IP from your address pool:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rook-ceph-dashboard-lb
  namespace: rook-ceph
  annotations:
    metallb.universe.tf/loadBalancerIPs: "192.168.1.200"
spec:
  type: LoadBalancer
  selector:
    app: rook-ceph-mgr
    rook_cluster: rook-ceph
  ports:
  - name: https-dashboard
    port: 443
    targetPort: 8443
    protocol: TCP
```

## Restrict Access with loadBalancerSourceRanges

Limit access to the dashboard to specific IP ranges for security:

```yaml
spec:
  type: LoadBalancer
  loadBalancerSourceRanges:
    - "10.0.0.0/8"
    - "192.168.1.0/24"
  selector:
    app: rook-ceph-mgr
    rook_cluster: rook-ceph
  ports:
  - name: https-dashboard
    port: 443
    targetPort: 8443
```

## Retrieve Dashboard Credentials

Get the admin password:

```bash
kubectl -n rook-ceph get secret rook-ceph-dashboard-password \
  -o jsonpath="{['data']['password']}" | base64 --decode && echo
```

Log in with username `admin` at `https://<external-ip>`.

## Verify TLS Certificate

The dashboard uses a self-signed certificate by default. To avoid browser warnings, either import the certificate or configure cert-manager. Check the current certificate:

```bash
openssl s_client -connect <external-ip>:443 -showcerts 2>/dev/null | \
  openssl x509 -noout -subject -dates
```

## Summary

Exposing the Ceph Dashboard via LoadBalancer provides a stable, cloud-native external IP that works seamlessly with cloud providers and MetalLB on bare-metal. Adding `loadBalancerSourceRanges` restricts access to trusted networks. Combined with a valid TLS certificate, this approach delivers a production-ready dashboard access path.
