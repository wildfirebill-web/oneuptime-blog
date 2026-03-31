# How to Debug Dapr Sidecar Startup Failures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Sidecar, Debugging, Startup, Kubernetes

Description: Learn how to diagnose and fix Dapr sidecar startup failures by analyzing init container logs, reviewing injection annotations, and checking operator connectivity.

---

## How Dapr Sidecar Injection Works on Kubernetes

Dapr uses a Kubernetes mutating admission webhook to inject the `daprd` sidecar container into pods at creation time. The Dapr operator runs in the `dapr-system` namespace and patches pod specs when it sees the `dapr.io/enabled: "true"` annotation. If injection fails, the pod starts without a sidecar and Dapr features are unavailable.

## Verifying Sidecar Injection

Check if the sidecar was injected by looking at pod containers:

```bash
kubectl get pod order-service-7b4c8d9f6-xk2pq -o jsonpath='{.spec.containers[*].name}'
```

Expected output: `order-service daprd`

If only `order-service` appears, the sidecar was not injected. Check the pod annotations:

```bash
kubectl get pod order-service-7b4c8d9f6-xk2pq -o jsonpath='{.metadata.annotations}' | jq
```

The required annotation must be present:

```json
{
  "dapr.io/enabled": "true",
  "dapr.io/app-id": "order-service",
  "dapr.io/app-port": "3000"
}
```

If the annotation is missing, update your deployment:

```yaml
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-service"
        dapr.io/app-port: "3000"
```

## Checking the Dapr Operator Logs

If annotations are correct but injection still fails, check the Dapr operator:

```bash
kubectl logs -n dapr-system -l app=dapr-operator --tail=50
```

Common operator errors:
- `Error patching pod` - RBAC issue or admission webhook misconfiguration
- `Failed to get certificate` - Dapr sentry is not running or TLS cert expired
- `connection refused` - Kubernetes API server connectivity issue

## Debugging Sidecar CrashLoopBackOff

If the sidecar container starts but crashes repeatedly:

```bash
kubectl describe pod order-service-7b4c8d9f6-xk2pq -n default | grep -A 20 "daprd"
```

Get the sidecar logs from the last crash:

```bash
kubectl logs order-service-7b4c8d9f6-xk2pq -c daprd --previous -n default
```

Common crash causes:

**App port not listening:**

```text
level=fatal msg="app health checks are failing" app_id=order-service
```

The app container is not listening on the configured `dapr.io/app-port`. Check your app is starting correctly:

```bash
kubectl logs order-service-7b4c8d9f6-xk2pq -c order-service -n default
```

**Placement service unreachable (Actors only):**

```text
level=error msg="error connecting to placement service" err="context deadline exceeded"
```

Check the placement service:

```bash
kubectl get pods -n dapr-system | grep placement
kubectl logs -n dapr-system -l app=dapr-placement
```

## Init Container Issues

Some Dapr configurations use an init container for certificate injection. Check init container status:

```bash
kubectl get pod order-service-pod -o jsonpath='{.status.initContainerStatuses}' | jq
```

If the init container is stuck:

```bash
kubectl logs order-service-pod -c dapr-init -n default
```

## Adjusting Resource Limits for the Sidecar

If the sidecar is OOMKilled, increase its resource limits in the Dapr configuration:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: dapr-config
spec:
  sidecarResourceRequests:
    cpu: "0.5"
    memory: "256Mi"
  sidecarResourceLimits:
    cpu: "1"
    memory: "512Mi"
```

Or set per-pod via annotations:

```yaml
annotations:
  dapr.io/sidecar-cpu-request: "0.5"
  dapr.io/sidecar-memory-request: "256Mi"
  dapr.io/sidecar-cpu-limit: "1"
  dapr.io/sidecar-memory-limit: "512Mi"
```

## Summary

Debugging Dapr sidecar startup failures follows a structured path: verify injection happened by checking containers, confirm annotations are correct, review Dapr operator logs for injection errors, and inspect the sidecar's own logs (including previous crash logs) for runtime failures. The most common issues are missing annotations, app containers not listening on the configured port, and insufficient resource limits causing OOMKill events.
