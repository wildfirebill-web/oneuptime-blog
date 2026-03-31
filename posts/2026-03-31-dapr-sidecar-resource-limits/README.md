# How to Configure Dapr Sidecar Resource Limits

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kubernetes, Resource, Sidecar, Performance

Description: Learn how to configure CPU and memory resource requests and limits for the Dapr sidecar container to optimize resource usage and prevent noisy-neighbor problems in production.

---

## Introduction

Every pod running Dapr has a `daprd` sidecar container injected alongside your application container. Without resource limits, the Dapr sidecar can consume unbounded CPU and memory, potentially starving your application. Configuring appropriate resource requests and limits for the Dapr sidecar is essential for predictable performance and cluster resource planning.

## Sidecar Resource Configuration Options

Dapr sidecar resources can be configured at three levels:

1. **Per-pod annotations** - override for a specific application
2. **Global default** - set via Helm values for all pods cluster-wide
3. **Dapr Configuration resource** - scoped to apps using that configuration

## Method 1: Per-Pod Annotations

Set resource requests and limits directly on your pod via annotations:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: default
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-service"
        dapr.io/app-port: "3000"
        # CPU limits for Dapr sidecar
        dapr.io/sidecar-cpu-limit: "500m"
        dapr.io/sidecar-cpu-request: "100m"
        # Memory limits for Dapr sidecar
        dapr.io/sidecar-memory-limit: "512Mi"
        dapr.io/sidecar-memory-request: "128Mi"
```

## Method 2: Global Defaults via Helm

Set defaults applied to all sidecars in the cluster:

```yaml
# dapr-values.yaml
dapr_sidecar_injector:
  defaultContainerConfig: |
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
```

Apply with Helm:

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --values dapr-values.yaml
```

## Method 3: ConfigMap for Sidecar Defaults

Dapr uses a ConfigMap in `dapr-system` for sidecar injection configuration:

```bash
kubectl get configmap dapr-config -n dapr-system -o yaml
```

Edit to set default resource constraints:

```bash
kubectl edit configmap dapr-config -n dapr-system
```

## Full Annotation Reference

All Dapr sidecar resource annotations:

```yaml
annotations:
  # Resource limits and requests
  dapr.io/sidecar-cpu-limit: "500m"
  dapr.io/sidecar-cpu-request: "100m"
  dapr.io/sidecar-memory-limit: "512Mi"
  dapr.io/sidecar-memory-request: "128Mi"

  # Liveness and readiness probe timeouts
  dapr.io/sidecar-liveness-probe-delay: "3"
  dapr.io/sidecar-liveness-probe-timeout: "3"
  dapr.io/sidecar-liveness-probe-period: "6"
  dapr.io/sidecar-liveness-probe-threshold: "3"
  dapr.io/sidecar-readiness-probe-delay: "3"
  dapr.io/sidecar-readiness-probe-timeout: "3"
  dapr.io/sidecar-readiness-probe-period: "6"
  dapr.io/sidecar-readiness-probe-threshold: "3"

  # Image and security
  dapr.io/sidecar-image: "daprio/daprd:1.14.0"
  dapr.io/sidecar-seccomp-profile-type: "RuntimeDefault"
```

## Sizing Guidelines

### Low-Traffic Services

```yaml
dapr.io/sidecar-cpu-request: "50m"
dapr.io/sidecar-cpu-limit: "200m"
dapr.io/sidecar-memory-request: "64Mi"
dapr.io/sidecar-memory-limit: "128Mi"
```

### Medium-Traffic Services

```yaml
dapr.io/sidecar-cpu-request: "100m"
dapr.io/sidecar-cpu-limit: "500m"
dapr.io/sidecar-memory-request: "128Mi"
dapr.io/sidecar-memory-limit: "256Mi"
```

### High-Traffic Services (with actors or workflows)

```yaml
dapr.io/sidecar-cpu-request: "250m"
dapr.io/sidecar-cpu-limit: "1000m"
dapr.io/sidecar-memory-request: "256Mi"
dapr.io/sidecar-memory-limit: "512Mi"
```

## Monitoring Sidecar Resource Usage

Check actual resource consumption:

```bash
# View resource usage across all sidecars
kubectl top pods --containers -A | grep daprd

# View resource usage for a specific app
kubectl top pods -l app=order-service --containers
```

Set up Prometheus alerts for sidecar resource saturation:

```yaml
groups:
- name: dapr-sidecar
  rules:
  - alert: DaprSidecarHighCPU
    expr: |
      rate(container_cpu_usage_seconds_total{container="daprd"}[5m]) > 0.8
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Dapr sidecar CPU usage high in {{ $labels.pod }}"

  - alert: DaprSidecarHighMemory
    expr: |
      container_memory_working_set_bytes{container="daprd"}
      / container_spec_memory_limit_bytes{container="daprd"} > 0.9
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Dapr sidecar memory near limit in {{ $labels.pod }}"
```

## Security Context for Sidecar

Also configure a security context to harden the sidecar:

```yaml
annotations:
  dapr.io/sidecar-drop-all-capabilities: "true"
  dapr.io/sidecar-seccomp-profile-type: "RuntimeDefault"
```

This drops all Linux capabilities and applies a seccomp profile to the Dapr sidecar.

## Vertical Pod Autoscaler (VPA) for Sidecars

Use VPA to automatically tune sidecar resource recommendations:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: order-service-vpa
  namespace: default
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  updatePolicy:
    updateMode: "Off"  # "Off" = recommend only, "Auto" = apply
  resourcePolicy:
    containerPolicies:
    - containerName: daprd
      maxAllowed:
        cpu: "1"
        memory: "512Mi"
      minAllowed:
        cpu: "50m"
        memory: "64Mi"
```

## Summary

Configure Dapr sidecar resource limits using pod annotations (`dapr.io/sidecar-cpu-limit`, `dapr.io/sidecar-memory-limit`) or globally via Helm values. Use the sizing guidelines as a starting point, then measure actual usage with `kubectl top pods --containers` and adjust based on real consumption. Set up Prometheus alerts for CPU and memory saturation, and consider using VPA in recommendation mode to get data-driven sizing suggestions. Always set both requests and limits to ensure the scheduler places pods correctly and prevents resource contention.
