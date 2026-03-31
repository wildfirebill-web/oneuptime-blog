# How to Troubleshoot Dapr Sidecar Resource Exhaustion

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Sidecar, Troubleshooting, Resource, Kubernetes

Description: Diagnose and resolve Dapr sidecar resource exhaustion including OOM kills, CPU throttling, and connection pool saturation that degrade application performance.

---

Resource exhaustion in the Dapr sidecar manifests as OOM kills, excessive CPU throttling, slow API responses, or connection timeouts. These issues often appear gradually under load rather than as sudden failures, making them harder to diagnose.

## Identifying the Problem

Start by checking if the sidecar is being OOM killed:

```bash
kubectl describe pod my-pod | grep -A 5 "OOMKilled"
kubectl get events --field-selector reason=OOMKilling --namespace production
```

Check for CPU throttling:

```bash
kubectl top pod my-pod --containers
```

If the daprd container consistently shows CPU near its limit, it is being throttled.

## Checking Current Resource Limits

View the actual resource limits set on the sidecar:

```bash
kubectl get pod my-pod -o json | \
  jq '.spec.containers[] | select(.name=="daprd") | .resources'
```

## Common Causes

**1. Too many concurrent requests**

The sidecar has a default request concurrency limit. Under high load, if connections pile up, memory grows.

Check the goroutine count via metrics:

```bash
curl http://localhost:9090/metrics | grep "go_goroutines"
```

If this number is very high (thousands), requests are backing up.

**2. Component connection pools filling up**

Redis or database connection pools may hold many open connections, each consuming memory.

```bash
kubectl logs my-pod -c daprd | grep -i "connection pool\|max connections\|timeout"
```

**3. Memory leak in a component**

Some component implementations may not release memory properly under certain workloads. Check if memory grows continuously over time using Prometheus:

```text
container_memory_working_set_bytes{container="daprd"}
```

## Increasing Resource Limits

If the sidecar legitimately needs more resources, increase the limits:

```yaml
annotations:
  dapr.io/sidecar-cpu-limit: "1000m"
  dapr.io/sidecar-memory-limit: "512Mi"
```

## Reducing Component Connection Overhead

Configure smaller connection pool sizes for state store components:

```yaml
spec:
  metadata:
    - name: maxConnections
      value: "10"
    - name: connectionPoolSize
      value: "5"
```

## Enabling Graceful Degradation

Configure resiliency policies to shed load before the sidecar becomes exhausted:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: my-resiliency
spec:
  policies:
    circuitBreakers:
      stateBreaker:
        maxRequests: 100
        trip: consecutiveFailures >= 10
        timeout: 30s
  targets:
    components:
      statestore:
        outbound:
          circuitBreaker: stateBreaker
```

## Summary

Troubleshooting Dapr sidecar resource exhaustion requires checking OOM events, CPU throttling metrics, connection pool usage, and memory growth trends. Combine resource limit increases with resiliency policies and connection pool tuning to prevent exhaustion under sustained load.
