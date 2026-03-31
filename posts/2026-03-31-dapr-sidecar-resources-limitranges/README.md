# How to Configure Dapr Sidecar Resources with Kubernetes LimitRanges

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kubernetes, LimitRange, Resource Management, Sidecar

Description: Configure resource requests and limits for Dapr sidecars using Kubernetes LimitRanges and per-pod annotations to ensure predictable resource consumption in production.

---

## Overview

Without resource limits, the Dapr sidecar (`daprd`) can consume unbounded CPU and memory, starving your application container. Kubernetes LimitRanges enforce default limits on all containers in a namespace, while Dapr annotations allow per-pod customization.

## Setting Dapr Sidecar Resources via Annotations

Configure CPU and memory for the Dapr sidecar on a specific deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-service"
        dapr.io/app-port: "3000"
        dapr.io/sidecar-cpu-request: "100m"
        dapr.io/sidecar-cpu-limit: "500m"
        dapr.io/sidecar-memory-request: "64Mi"
        dapr.io/sidecar-memory-limit: "256Mi"
```

## Namespace-Level LimitRange for Dapr Sidecars

Apply a LimitRange to set defaults for all containers in the namespace:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dapr-namespace-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "64Mi"
    max:
      cpu: "2"
      memory: "1Gi"
    min:
      cpu: "50m"
      memory: "32Mi"
```

Apply the LimitRange:

```bash
kubectl apply -f limitrange.yaml
kubectl describe limitrange dapr-namespace-limits -n production
```

## Sizing Dapr Sidecar Resources

Use these guidelines based on workload type:

```yaml
# Low-traffic service (< 100 req/s)
dapr.io/sidecar-cpu-request: "50m"
dapr.io/sidecar-cpu-limit: "200m"
dapr.io/sidecar-memory-request: "32Mi"
dapr.io/sidecar-memory-limit: "128Mi"

# Medium-traffic service (100-1000 req/s)
dapr.io/sidecar-cpu-request: "100m"
dapr.io/sidecar-cpu-limit: "500m"
dapr.io/sidecar-memory-request: "64Mi"
dapr.io/sidecar-memory-limit: "256Mi"

# High-traffic service (> 1000 req/s)
dapr.io/sidecar-cpu-request: "500m"
dapr.io/sidecar-cpu-limit: "2000m"
dapr.io/sidecar-memory-request: "256Mi"
dapr.io/sidecar-memory-limit: "1Gi"
```

## Monitoring Sidecar Resource Usage

Check actual resource consumption of Dapr sidecars:

```bash
# Get resource usage for all daprd containers
kubectl top pods --containers -n production | grep daprd

# Check resource requests/limits on the sidecar container
kubectl get pod order-service-abc -o jsonpath='{.spec.containers[?(@.name=="daprd")].resources}'
```

## Handling LimitRange Admission Errors

If a deployment fails due to LimitRange violations, you'll see:

```bash
# Error example
kubectl describe replicaset order-service-xyz
# Error: pods "order-service-xyz" is forbidden:
# maximum cpu usage per Container is 2, but limit is 4.

# Fix by adjusting sidecar resource annotations
kubectl patch deployment order-service --type=json \
  -p='[{"op":"replace","path":"/spec/template/metadata/annotations/dapr.io~1sidecar-cpu-limit","value":"2000m"}]'
```

## Summary

Properly sizing Dapr sidecar resources prevents both resource starvation and overprovisioning. Use per-pod annotations to tune sidecar resources for high-traffic services, and rely on namespace LimitRanges to enforce sensible defaults and ceilings for all services. Monitor actual consumption with `kubectl top` and adjust limits as traffic patterns evolve.
