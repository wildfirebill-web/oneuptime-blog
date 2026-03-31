# How to Fix Dapr Pod OOMKilled Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kubernetes, Memory, OOMKilled, Resource Management

Description: Diagnose and fix OOMKilled errors in Dapr sidecar containers by tuning memory limits, configuring resource requests, and identifying memory leaks.

---

OOMKilled (Out of Memory Killed) errors in Dapr pods mean the `daprd` sidecar or your application container is exceeding its memory limit. Kubernetes terminates the container and the pod restarts.

## Identifying OOMKilled Events

Check pod restart counts and exit reasons:

```bash
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace> | grep -A5 "Last State"
```

An OOMKilled container shows:

```text
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

Also check events:

```bash
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | grep OOM
```

## Default Dapr Sidecar Memory Limits

By default, Dapr injects sidecars with 256Mi memory limit, which may be insufficient for high-throughput applications. Override via annotations:

```yaml
annotations:
  dapr.io/sidecar-memory-limit: "512Mi"
  dapr.io/sidecar-memory-request: "256Mi"
  dapr.io/sidecar-cpu-limit: "500m"
  dapr.io/sidecar-cpu-request: "250m"
```

## Tuning via Helm Values

Set default resource limits for all injected sidecars in the Helm chart:

```yaml
# values.yaml
dapr_sidecar_injector:
  injectorResources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
```

```bash
helm upgrade dapr dapr/dapr -n dapr-system -f values.yaml --reuse-values
```

## Profiling Memory Usage

Enable the Dapr profiling endpoint to capture memory profiles:

```yaml
annotations:
  dapr.io/enable-profiling: "true"
```

Then capture a heap profile:

```bash
kubectl port-forward <pod-name> 7777:7777
curl http://localhost:7777/debug/pprof/heap > heap.out
go tool pprof heap.out
```

## Common Memory Leak Sources

**Large message payloads:** Dapr buffers messages in memory. If pub/sub messages are very large, memory grows quickly. Use smaller payloads or streaming.

**Actor state size:** If actors store large objects in state, memory usage scales with actor count. Minimize state per actor.

**Metrics cardinality:** High-cardinality labels in telemetry can cause memory bloat. Reduce custom labels:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
spec:
  metric:
    enabled: true
    rules:
    - name: dapr_service_invocation_req_sent_total
      labels:
      - name: app_id
        regex: {}
```

## Configuring Actor Memory Thresholds

For actor-heavy workloads, tune the actor scan interval to garbage collect inactive actors sooner:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
spec:
  features:
  - name: ActorStateTTL
    enabled: true
```

## Summary

Dapr OOMKilled issues are resolved by increasing sidecar memory limits via annotations, profiling the sidecar to find leaks, and reducing memory pressure from large payloads, actor state, or high-cardinality metrics. Set memory requests and limits explicitly on every pod to give Kubernetes accurate scheduling information.
