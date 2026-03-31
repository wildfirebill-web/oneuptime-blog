# How to Configure Network Policies for Dapr on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kubernetes, Network Policy, Security, Networking

Description: Learn how to configure Kubernetes NetworkPolicies to control traffic to and from Dapr sidecars and control plane components for defense in depth.

---

## Overview

Dapr's mTLS secures application-layer communication, but Kubernetes NetworkPolicies add a network-layer enforcement that blocks unwanted TCP connections before they even reach the sidecar. Combining both provides defense in depth and is required by many security compliance frameworks.

## Dapr Ports to Know

Before writing policies, understand the default ports Dapr uses:

| Component | Port | Protocol |
|---|---|---|
| Sidecar HTTP API | 3500 | HTTP |
| Sidecar gRPC API | 50001 | gRPC |
| Sidecar internal | 50002 | gRPC |
| Sidecar metrics | 9090 | HTTP |
| Sentry (CA) | 50001 | gRPC |
| Operator | 6500 | gRPC |
| Placement | 50005 | gRPC |
| Dashboard | 8080 | HTTP |

## Allow App-to-Sidecar Traffic Only

Restrict the Dapr HTTP port so only the application container (on the same pod) can reach it. In Kubernetes, containers in the same pod share a network namespace, so this policy blocks external pods from reaching the sidecar API:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-dapr-sidecar
  namespace: default
spec:
  podSelector:
    matchLabels:
      dapr.io/enabled: "true"
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          dapr.io/enabled: "true"
    ports:
    - protocol: TCP
      port: 3500
    - protocol: TCP
      port: 50001
```

## Allow Dapr Control Plane to Reach Sidecars

The Dapr operator and sentry need to communicate with sidecars. Allow traffic from the `dapr-system` namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dapr-control-plane
  namespace: default
spec:
  podSelector:
    matchLabels:
      dapr.io/enabled: "true"
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: dapr-system
    ports:
    - protocol: TCP
      port: 50001
    - protocol: TCP
      port: 50002
```

## Deny All by Default

Start with a default-deny policy and build up explicit allows:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

## Allow Sidecar Egress to Dependencies

If services use Dapr bindings to call external APIs, allow egress to specific CIDRs or ports:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dapr-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      dapr.io/enabled: "true"
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: dapr-system
  - to:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 6379
    - protocol: TCP
      port: 5432
```

## Testing Network Policies

Use `kubectl exec` to test connectivity:

```bash
kubectl run test-pod --image=busybox --restart=Never -it --rm \
  -- wget -qO- http://order-service-pod-ip:3500/v1.0/metadata
# Should fail if NetworkPolicy blocks cross-pod sidecar access
```

## Summary

Kubernetes NetworkPolicies complement Dapr mTLS by enforcing network-layer access control. A default-deny baseline combined with explicit allows for Dapr control plane traffic and inter-service communication ensures that even if mTLS is misconfigured, unauthorized pods cannot establish connections to your sidecars.
