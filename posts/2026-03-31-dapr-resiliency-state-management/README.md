# How to Apply Resiliency Policies to State Management in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Resiliency, State Management, Retry, Fault Tolerance

Description: Learn how to apply Dapr resiliency policies to state store operations to handle transient database failures with automatic retries and circuit breakers.

---

## Overview

State store operations can fail due to network issues, temporary database overload, or rolling infrastructure updates. Dapr resiliency policies applied to state management components add automatic retry and circuit breaker protection to every `get`, `set`, `delete`, and `transaction` operation without any changes to your application code.

## Configuring Resiliency for State Stores

State store resiliency is configured under `targets.components` with an `outbound` section (state operations are always outbound from your app to the store):

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: state-resiliency
  namespace: default
spec:
  policies:
    timeouts:
      stateTimeout: 5s
      criticalStateTimeout: 2s
    retries:
      stateRetry:
        policy: exponential
        initialInterval: 500ms
        multiplier: 2.0
        maxInterval: 30s
        maxRetries: 5
        randomizationFactor: 0.5
      criticalRetry:
        policy: constant
        duration: 200ms
        maxRetries: 3
    circuitBreakers:
      stateCB:
        maxRequests: 1
        interval: 10s
        timeout: 30s
        trip: consecutiveFailures >= 5
  targets:
    components:
      my-redis-state:
        outbound:
          timeout: stateTimeout
          retry: stateRetry
          circuitBreaker: stateCB
      session-state:
        outbound:
          timeout: criticalStateTimeout
          retry: criticalRetry
```

## Application Code Remains Unchanged

Your state management calls are identical whether or not a resiliency policy is active:

```python
from dapr.clients import DaprClient
import json

with DaprClient() as d:
    # Save state - resiliency policy applies automatically
    d.save_state(
        store_name='my-redis-state',
        key='user-123',
        value=json.dumps({'name': 'Alice', 'balance': 500})
    )

    # Get state - also protected by resiliency
    state = d.get_state(store_name='my-redis-state', key='user-123')
    print(state.data)
```

## Transactional State with Resiliency

Bulk and transactional state operations also benefit from the same policies:

```javascript
const { DaprClient, HttpMethod } = require('@dapr/dapr');
const client = new DaprClient();

// Multi-set - all operations protected by the state resiliency policy
await client.state.save('my-redis-state', [
  { key: 'order-1', value: { status: 'pending' } },
  { key: 'order-2', value: { status: 'pending' } }
]);
```

## State Store Circuit Breaker Behavior

When the state store circuit breaker trips, state operations immediately fail without attempting the network call. Your application should handle this gracefully:

```javascript
try {
  const state = await client.state.get('my-redis-state', 'user-123');
  return state;
} catch (err) {
  // Circuit is open - use a fallback
  console.warn('State store unavailable, using fallback');
  return getFromInMemoryCache('user-123');
}
```

## Applying Policies to Multiple State Stores

```yaml
targets:
  components:
    default:
      outbound:
        timeout: stateTimeout
        retry: stateRetry
    critical-state:
      outbound:
        timeout: criticalStateTimeout
        retry: criticalRetry
        circuitBreaker: stateCB
```

The `default` target applies to all components not explicitly listed.

## Monitoring State Resiliency

```bash
# Check retry events in sidecar logs
kubectl logs deployment/my-app -c daprd \
  | grep -E "state|retry|circuit" | tail -20

# Prometheus metrics for state operations
curl http://localhost:9090/metrics \
  | grep "dapr_component_state"
```

## Summary

Applying Dapr resiliency policies to state management components ensures that transient database failures are automatically handled through configurable retry and circuit breaker rules. The policies require no application code changes and protect all state operations including gets, saves, deletes, and transactions.
