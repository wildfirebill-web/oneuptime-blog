# How to Implement Tenant-Specific Resiliency Policies with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Resiliency, Multi-Tenancy, Retry, Circuit Breaker

Description: Implement tenant-specific Dapr resiliency policies using namespace-scoped Resiliency CRDs to provide differentiated SLAs with custom retry, timeout, and circuit breaker settings.

---

## Why Tenant-Specific Resiliency Policies

Different tenants often have different reliability requirements. A premium tenant may need aggressive retries and longer timeouts to meet a strict SLA, while a free-tier tenant might use conservative circuit breakers to prevent cascading failures. Dapr's Resiliency CRD, scoped to a namespace, enables this differentiation without changing application code.

## Creating Namespace-Scoped Resiliency CRDs

Deploy a Resiliency resource in each tenant namespace:

```yaml
# Tenant A - premium plan, aggressive retries
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: resiliency
  namespace: tenant-a
spec:
  policies:
    timeouts:
      defaultTimeout: 10s
      stateTimeout: 5s
    retries:
      defaultRetry:
        policy: exponential
        duration: 500ms
        maxInterval: 15s
        maxRetries: 10
      stateRetry:
        policy: constant
        duration: 200ms
        maxRetries: 5
    circuitBreakers:
      defaultCB:
        maxRequests: 5
        interval: 30s
        timeout: 30s
        trip: consecutiveFailures >= 10
  targets:
    apps:
      orders-api:
        timeout: defaultTimeout
        retry: defaultRetry
        circuitBreaker: defaultCB
    components:
      statestore:
        outbound:
          timeout: stateTimeout
          retry: stateRetry
```

```yaml
# Tenant B - free plan, conservative settings
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: resiliency
  namespace: tenant-b
spec:
  policies:
    timeouts:
      defaultTimeout: 3s
    retries:
      defaultRetry:
        policy: constant
        duration: 1s
        maxRetries: 3
    circuitBreakers:
      defaultCB:
        maxRequests: 1
        interval: 60s
        timeout: 60s
        trip: consecutiveFailures >= 3
  targets:
    apps:
      orders-api:
        timeout: defaultTimeout
        retry: defaultRetry
        circuitBreaker: defaultCB
```

## Applying Resiliency to All Outbound Calls

Use the `actors` and `components` targets to cover all outbound operations:

```yaml
spec:
  targets:
    components:
      statestore:
        outbound:
          retry: defaultRetry
          timeout: defaultTimeout
      pubsub:
        outbound:
          retry: defaultRetry
```

## Verifying Resiliency Policies Are Active

Check that the Resiliency CRD is loaded correctly:

```bash
kubectl get resiliency -n tenant-a
kubectl describe resiliency resiliency -n tenant-a
```

The sidecar logs confirm loading:

```bash
kubectl logs my-pod -c daprd | grep "resiliency"
# resiliency configuration loaded: resiliency
```

## Testing Resiliency Behavior

Simulate a backend failure and verify retries occur:

```bash
# Scale down the state store backend
kubectl scale deployment redis -n tenant-a --replicas=0

# Make a state call - should see retries in logs
curl -X POST http://localhost:3500/v1.0/state/statestore \
  -H "Content-Type: application/json" \
  -d '[{"key": "test", "value": "data"}]'

kubectl logs my-pod -c daprd | grep "retry\|circuit"
```

## Summary

Namespace-scoped Dapr Resiliency CRDs allow per-tenant SLA differentiation without application code changes. Premium tenants receive aggressive retry and timeout policies while free-tier tenants use conservative circuit breakers to protect shared infrastructure. The consistent CRD name across namespaces means application annotations reference `resiliency` uniformly while each tenant's pod picks up the namespace-local policy.
