# How to Use Dapr and Linkerd Together

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Linkerd, Service Mesh, Integration, Kubernetes

Description: Configure Dapr and Linkerd to work together on Kubernetes, enabling Linkerd's lightweight mTLS and observability alongside Dapr's application building blocks.

---

Dapr and Linkerd are a natural pairing for teams that want Dapr's application-level building blocks combined with Linkerd's lightweight, zero-config service mesh capabilities. This guide walks through the configuration steps to make them work together correctly.

## Why Combine Dapr and Linkerd

**Dapr provides:** State management, pub/sub, service invocation, actors, workflows, bindings.

**Linkerd provides:** Automatic mTLS between all meshed services, latency-aware load balancing, golden metrics (success rate, requests per second, latency), and traffic splitting with zero application changes.

Together, Linkerd handles the network layer while Dapr handles the application layer.

## Installation

Install Linkerd first, then Dapr:

```bash
# Install Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh

# Install Linkerd control plane
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -
linkerd check

# Install Dapr
helm install dapr dapr/dapr -n dapr-system --create-namespace --wait
```

## Meshing the dapr-system Namespace

Inject Linkerd into the Dapr control plane:

```bash
kubectl annotate namespace dapr-system \
  linkerd.io/inject=enabled
kubectl rollout restart deployment -n dapr-system
```

## Meshing Your Application Namespace

```bash
kubectl annotate namespace default \
  linkerd.io/inject=enabled
```

Or per-deployment:

```yaml
annotations:
  linkerd.io/inject: "enabled"
  dapr.io/enabled: "true"
  dapr.io/app-id: "myapp"
```

## Disabling Dapr's mTLS

Since Linkerd handles mTLS for all pod-to-pod traffic, disable Dapr's built-in mTLS to avoid conflicts:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: dapr-config
  namespace: default
spec:
  mtls:
    enabled: false
```

```yaml
annotations:
  dapr.io/config: "dapr-config"
```

## Handling Port Exclusions

Linkerd should not intercept Dapr's internal health check and metrics ports. Annotate your pods:

```yaml
annotations:
  config.linkerd.io/skip-inbound-ports: "3500,50001,50002,9090"
  config.linkerd.io/skip-outbound-ports: "3500,50001,50002"
```

## Viewing Linkerd Metrics for Dapr Traffic

Dapr service invocation calls appear in Linkerd's observability as regular HTTP/gRPC traffic:

```bash
# Install Linkerd viz
linkerd viz install | kubectl apply -f -
linkerd viz dashboard
```

In the dashboard, you will see success rates and latency for all Dapr-mediated service calls.

## Traffic Splitting with Linkerd

Use Linkerd's TrafficSplit for canary releases of Dapr-enabled services:

```yaml
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: order-service-split
spec:
  service: order-service
  backends:
  - service: order-service-v1
    weight: 90
  - service: order-service-v2
    weight: 10
```

## Summary

Dapr and Linkerd work well together with two key configuration steps: disable Dapr's built-in mTLS to let Linkerd handle encryption, and exclude Dapr's internal ports from Linkerd's proxy interception. Linkerd's lightweight proxy adds virtually no overhead while providing automatic mTLS and golden metrics for all Dapr-mediated service calls.
