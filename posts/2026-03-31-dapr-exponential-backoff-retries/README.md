# How to Use Exponential Backoff Retries in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Resiliency, Retry, Exponential Backoff, Fault Tolerance

Description: Learn how to configure exponential backoff retry policies in Dapr to gracefully handle transient failures while reducing load on struggling downstream services.

---

## Overview

Exponential backoff increases the wait time between each retry attempt, typically multiplying by a factor after each failure. This reduces pressure on an already-struggling service and prevents thundering herd scenarios where many callers simultaneously hammer a recovering dependency. Dapr's `exponential` retry policy supports full customization of growth rate, maximum interval, and jitter.

## Configuring Exponential Backoff

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: exponential-retry-resiliency
  namespace: default
spec:
  policies:
    retries:
      standardExponential:
        policy: exponential
        initialInterval: 500ms
        multiplier: 2.0
        maxInterval: 30s
        maxRetries: 8
        randomizationFactor: 0.5
```

With `initialInterval: 500ms` and `multiplier: 2.0`, the retry intervals grow as follows (before jitter):

| Attempt | Interval |
|---|---|
| 1 | 500ms |
| 2 | 1s |
| 3 | 2s |
| 4 | 4s |
| 5 | 8s |
| 6 | 16s |
| 7 | 30s (capped) |
| 8 | 30s (capped) |

## The Role of randomizationFactor

`randomizationFactor` adds jitter to each computed interval. With a factor of `0.5`, each interval is multiplied by a random value between `0.5` and `1.5`. This spreads retry timing across callers and avoids synchronized retry storms:

```yaml
retries:
  jitteredExponential:
    policy: exponential
    initialInterval: 1s
    multiplier: 1.5
    maxInterval: 60s
    maxRetries: 10
    randomizationFactor: 0.5
```

## Applying to Services and Components

```yaml
targets:
  apps:
    payment-gateway:
      retry: standardExponential
    notification-service:
      retry: jitteredExponential
  components:
    postgres-state:
      outbound:
        retry: standardExponential
```

## Exponential Backoff for Pub/Sub Redelivery

When a message consumer fails, Dapr can retry delivery using exponential backoff:

```yaml
targets:
  components:
    orders-kafka:
      inbound:
        retry: jitteredExponential
```

This means if the subscriber crashes during message processing, Dapr backs off before redelivering, giving the service time to recover.

## Calculating Max Total Time

With `maxRetries: 8`, `initialInterval: 500ms`, `multiplier: 2.0`, and `maxInterval: 30s`, the worst-case total retry time (before jitter) is approximately:

```
500ms + 1s + 2s + 4s + 8s + 16s + 30s + 30s = 91.5s
```

Add the per-attempt timeout to get the absolute worst case. Plan your upstream timeouts and SLAs accordingly.

## Real-World Example: Database Reconnection

```yaml
retries:
  dbReconnect:
    policy: exponential
    initialInterval: 1s
    multiplier: 2.0
    maxInterval: 30s
    maxRetries: -1
    randomizationFactor: 0.3

targets:
  components:
    primary-database:
      outbound:
        retry: dbReconnect
```

Using `maxRetries: -1` with exponential backoff is safe for database connections because the interval grows to `maxInterval` and stays there, rather than retrying indefinitely at high frequency.

## Summary

Dapr's exponential backoff retry policy increases wait time between retries with configurable multiplier, cap, and jitter. It is the recommended strategy for most production scenarios because it reduces load on recovering services and naturally prevents synchronized retry bursts from multiple callers.
