# How to Configure Retry Policies in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Resiliency, Retry, Policy, Fault Tolerance

Description: Learn how to configure Dapr retry policies in Resiliency resources to automatically retry failed service calls and component operations with customizable backoff strategies.

---

## Overview

Transient failures are common in distributed systems: network blips, temporary resource contention, and rolling restarts all cause intermittent errors. Dapr retry policies automatically retry failed operations according to rules you define, making your services resilient to these transient faults without adding retry logic to your application code.

## Retry Policy Basics

Retry policies are defined in the `policies.retries` section of a `Resiliency` resource. Each policy has a name, a retry strategy (`constant` or `exponential`), and limits on retries:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: app-resiliency
  namespace: default
spec:
  policies:
    retries:
      constantRetry:
        policy: constant
        duration: 1s
        maxRetries: 5
      exponentialRetry:
        policy: exponential
        initialInterval: 500ms
        maxInterval: 60s
        multiplier: 1.5
        maxRetries: 10
        randomizationFactor: 0.5
```

## Constant Retry Policy Fields

| Field | Description |
|---|---|
| `policy` | Must be `constant` |
| `duration` | Wait between each retry |
| `maxRetries` | Maximum number of retry attempts (-1 for unlimited) |

## Exponential Retry Policy Fields

| Field | Description |
|---|---|
| `policy` | Must be `exponential` |
| `initialInterval` | First wait duration |
| `multiplier` | Multiplier applied to each subsequent interval |
| `maxInterval` | Cap on the wait duration |
| `maxRetries` | Maximum retry attempts |
| `randomizationFactor` | Jitter factor (0 to 1) to spread retry storms |

## Applying Retries to Services and Components

```yaml
targets:
  apps:
    inventory-service:
      retry: constantRetry
    analytics-service:
      retry: exponentialRetry
  components:
    postgres-state:
      outbound:
        retry: exponentialRetry
```

## What Gets Retried

Dapr retries operations that fail with retryable errors. For HTTP, these include `429 Too Many Requests`, `502 Bad Gateway`, `503 Service Unavailable`, and `504 Gateway Timeout`. For gRPC, retryable codes include `UNAVAILABLE` and `RESOURCE_EXHAUSTED`.

Non-retryable errors (like `400 Bad Request` or `404 Not Found`) are not retried because retrying a bad request will not succeed.

## Retry with Jitter to Avoid Thundering Herd

When many services fail simultaneously and retry at the same interval, they can overwhelm the recovering service. Use `randomizationFactor` to spread retries:

```yaml
retries:
  jitteredRetry:
    policy: exponential
    initialInterval: 1s
    maxInterval: 30s
    multiplier: 2.0
    maxRetries: 8
    randomizationFactor: 0.5
```

With `randomizationFactor: 0.5`, each retry interval is randomized by up to 50% in either direction.

## Observing Retries in Logs

```bash
kubectl logs deployment/order-service -c daprd \
  | grep -i retry | tail -20
```

You should see log entries like: `Retry attempt 1 for app target inventory-service`.

## Summary

Dapr retry policies declaratively add automatic retry behavior to service invocations and component operations. Use constant retries for quick, fixed-interval retries and exponential backoff with jitter for resilient retry behavior under load. Setting `maxRetries` prevents infinite retry loops that can worsen cascading failures.
