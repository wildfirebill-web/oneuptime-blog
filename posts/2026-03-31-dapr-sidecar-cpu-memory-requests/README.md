# How to Configure Dapr Sidecar CPU and Memory Requests

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Sidecar, Kubernetes, Resource, Configuration

Description: Set appropriate CPU and memory requests and limits for the Dapr sidecar container to ensure stable scheduling and prevent resource starvation in production clusters.

---

Every Dapr-enabled pod runs two containers: your application and the daprd sidecar. If you do not set resource requests and limits on the sidecar, the Kubernetes scheduler may place pods on nodes without enough capacity, leading to OOM kills or CPU throttling that degrades throughput.

## Default Sidecar Resources

By default, the Dapr injector does not set resource requests or limits for the sidecar container. This means the sidecar competes with other containers for node resources without any guarantees.

## Setting Resource Requests via Annotations

The simplest way to configure sidecar resources in Kubernetes is with pod annotations:

```yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "payment-service"
  dapr.io/sidecar-cpu-request: "100m"
  dapr.io/sidecar-memory-request: "128Mi"
  dapr.io/sidecar-cpu-limit: "500m"
  dapr.io/sidecar-memory-limit: "256Mi"
```

These annotations map directly to the Kubernetes resource spec injected into the sidecar container.

## What Gets Injected

The injector translates these annotations into a standard container resources block:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

## Choosing Appropriate Values

A typical daprd sidecar uses:
- CPU: 50-200m at idle, spikes to 500m+ under heavy request load
- Memory: 64-150Mi at idle, higher with many components or actors

For services with light traffic:

```yaml
dapr.io/sidecar-cpu-request: "50m"
dapr.io/sidecar-memory-request: "64Mi"
dapr.io/sidecar-cpu-limit: "250m"
dapr.io/sidecar-memory-limit: "128Mi"
```

For services with heavy traffic or many components:

```yaml
dapr.io/sidecar-cpu-request: "200m"
dapr.io/sidecar-memory-request: "256Mi"
dapr.io/sidecar-cpu-limit: "1000m"
dapr.io/sidecar-memory-limit: "512Mi"
```

## Setting Global Defaults with Helm

You can set default sidecar resources globally when installing or upgrading the Dapr Helm chart:

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --set dapr_sidecar_injector.defaultConfig.cpuRequest=100m \
  --set dapr_sidecar_injector.defaultConfig.memoryRequest=128Mi \
  --set dapr_sidecar_injector.defaultConfig.cpuLimit=500m \
  --set dapr_sidecar_injector.defaultConfig.memoryLimit=256Mi
```

## Monitor Actual Usage

Use kubectl top to observe real sidecar resource usage and tune your requests accordingly:

```bash
kubectl top pod my-pod --containers
```

Or query Prometheus for sidecar resource metrics:

```bash
rate(container_cpu_usage_seconds_total{container="daprd"}[5m])
```

## Summary

Setting CPU and memory requests and limits on the Dapr sidecar ensures predictable scheduling, prevents resource starvation, and enables the cluster autoscaler to make accurate scaling decisions. Start with conservative values, observe actual usage, and tune based on your workload profile.
