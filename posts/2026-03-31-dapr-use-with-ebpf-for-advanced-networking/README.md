# How to Use Dapr with eBPF for Advanced Networking

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, eBPF, Networking, Observability, Cilium

Description: Learn how eBPF-based tools like Cilium complement Dapr by providing kernel-level observability, network policy enforcement, and service identity without additional sidecars.

---

Dapr and eBPF tools like Cilium operate at different layers of the networking stack. Understanding how they complement each other lets you build more observable and secure microservice platforms.

## Where Dapr and eBPF Fit

Dapr operates at the application layer - it provides building blocks for state, messaging, and service invocation via the sidecar pattern. eBPF tools like Cilium operate at the kernel network layer, intercepting network traffic without requiring application code changes or additional proxy sidecars.

```
Application Code
     |
Dapr Sidecar (L7 - application building blocks)
     |
Cilium eBPF (L3/L4/L7 - network policy, identity, observability)
     |
Kernel Network Stack
```

## Cilium with Dapr on Kubernetes

Install Cilium as the Kubernetes CNI alongside Dapr:

```bash
# Install Cilium
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.16.0 \
  --namespace kube-system \
  --set kubeProxyReplacement=true

# Install Dapr
dapr init -k

# Verify both are running
kubectl get pods -n kube-system | grep cilium
dapr status -k
```

## Network Policy for Dapr Services

Use Cilium NetworkPolicy to restrict which services can communicate through Dapr:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: dapr-service-policy
spec:
  endpointSelector:
    matchLabels:
      app: order-processor
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: checkout-service
      toPorts:
        - ports:
            - port: "3500"    # Dapr sidecar HTTP port
              protocol: TCP
```

This allows only `checkout-service` to reach `order-processor`'s Dapr sidecar, blocking all other sources at the network layer independently of Dapr's mTLS.

## eBPF-Based Observability for Dapr Traffic

Use Hubble (Cilium's observability layer) to trace Dapr sidecar traffic at the network level:

```bash
# Install Hubble
cilium hubble enable

# Watch live Dapr traffic
hubble observe --namespace production --follow \
  --port 3500 \
  --output json | jq '{src: .source.namespace, dst: .destination.namespace, verdict: .verdict}'
```

This shows every Dapr sidecar-to-sidecar call at the kernel level, complementing Dapr's distributed traces.

## eBPF Bandwidth Limiting for Dapr Services

Use Cilium's bandwidth manager to throttle noisy Dapr services:

```yaml
apiVersion: cilium.io/v2
kind: CiliumBandwidthManager
metadata:
  name: order-processor-bwm
spec:
  endpointSelector:
    matchLabels:
      app: order-processor
  bandwidth:
    egress:
      rate: "100Mbit"
```

This prevents a single service's Dapr traffic from saturating the cluster network.

## Mutual Authentication Layers

Dapr provides mTLS between sidecars at the application layer. Cilium adds a separate mutual authentication layer at the network layer using SPIFFE/SPIRE identity:

```yaml
# Cilium mutual auth policy
spec:
  authentication:
    mode: "required"
    spire:
      agentSocketPath: "/run/spire/sockets/agent.sock"
```

Running both provides defense-in-depth: even if Dapr mTLS is misconfigured, Cilium's network-level identity still blocks unauthorized connections.

## Summary

Dapr and eBPF tools like Cilium complement each other at different layers. Dapr handles application-level building blocks with mTLS, while Cilium provides kernel-level network policy, identity enforcement, and traffic observability via Hubble. Together they provide defense-in-depth security and comprehensive visibility for Dapr-based microservices.
