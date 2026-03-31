# How to Plan Dapr Capacity for Production Workloads

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Capacity Planning, Performance, Kubernetes, Resource

Description: Learn how to estimate and configure resource requests for Dapr sidecars and control plane components to right-size your production cluster for expected workloads.

---

Capacity planning for Dapr involves sizing both the control plane components and the per-application sidecar proxies. The sidecar overhead is often the most impactful factor since every application pod runs one.

## Dapr Sidecar Resource Baseline

Each Dapr sidecar (`daprd`) consumes resources based on message throughput and the number of active subscriptions. A typical baseline for a moderate-traffic service:

```yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "order-service"
  dapr.io/app-port: "8080"
  dapr.io/sidecar-cpu-request: "100m"
  dapr.io/sidecar-cpu-limit: "500m"
  dapr.io/sidecar-memory-request: "64Mi"
  dapr.io/sidecar-memory-limit: "256Mi"
```

For high-throughput services (1000+ req/s), increase the CPU limit to 1000m and memory to 512Mi.

## Estimating Total Sidecar Overhead

Calculate total cluster overhead from sidecars:

```bash
# Number of application pods
APP_PODS=50

# Sidecar requests per pod
SIDECAR_CPU_M=100    # millicores
SIDECAR_MEM_MI=64    # MiB

# Total overhead
TOTAL_CPU=$((APP_PODS * SIDECAR_CPU_M))  # = 5000m = 5 vCPU
TOTAL_MEM=$((APP_PODS * SIDECAR_MEM_MI)) # = 3200 MiB

echo "Total sidecar CPU requests: ${TOTAL_CPU}m"
echo "Total sidecar memory requests: ${TOTAL_MEM}Mi"
```

## Control Plane Resource Requirements

For a cluster with 50-100 application pods, the control plane components need:

```yaml
dapr_operator:
  replicaCount: 3
  resources:
    requests:
      cpu: 100m
      memory: 200Mi
    limits:
      cpu: 1000m
      memory: 1Gi

dapr_placement:
  replicaCount: 3
  resources:
    requests:
      cpu: 250m
      memory: 500Mi
    limits:
      cpu: 2000m
      memory: 2Gi
```

The placement service is the most resource-intensive component when using Dapr actors at scale.

## Load Testing to Validate Capacity

Use k6 or hey to generate load and observe Dapr metrics:

```bash
# Install hey
go install github.com/rakyll/hey@latest

# Run load test against a Dapr-enabled service
hey -n 10000 -c 100 \
  http://order-service.default.svc.cluster.local:80/orders

# Watch Dapr sidecar resource usage during test
kubectl top pods -l app=order-service --containers
```

## Horizontal Pod Autoscaling with Dapr

Scale application pods based on pub/sub queue depth using KEDA alongside Dapr:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 2
  maxReplicaCount: 20
  triggers:
  - type: redis
    metadata:
      address: redis:6379
      listName: "order-processor||orders"
      listLength: "50"
```

## Capacity Planning for Actors

For Dapr actor workloads, the placement service partitions actors across hosts. Plan for the placement service memory to scale with actor count:

```bash
# Rough formula: 1 KB per active actor
ACTIVE_ACTORS=100000
PLACEMENT_MEM_MB=$((ACTIVE_ACTORS / 1024))
echo "Placement memory estimate: ${PLACEMENT_MEM_MB}Mi"
```

## Summary

Dapr capacity planning starts with sizing sidecar resources based on per-service throughput, then aggregating across all pods to determine cluster overhead. Control plane sizing depends on application count and actor usage. Load testing under realistic traffic patterns reveals actual resource consumption before production deployment.
