# How to Understand Dapr Resiliency Specification Format

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Resiliency, Retry, Circuit Breaker, Timeout, Specification

Description: Understand the Dapr resiliency specification format to configure retries, timeouts, and circuit breakers for service invocation, state stores, and pub/sub.

---

## What Is a Dapr Resiliency Policy?

Dapr resiliency policies define how the Dapr sidecar handles transient failures in service calls, state operations, and pub/sub. Instead of baking retry logic into application code, you declare it in a YAML file and attach it to targets.

## Basic Resiliency Spec Structure

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: myresiliency
spec:
  policies:
    retries:
      standard-retry:
        policy: constant
        duration: 1s
        maxRetries: 3

    timeouts:
      standard-timeout: 5s

    circuitBreakers:
      standard-cb:
        maxRequests: 1
        interval: 10s
        timeout: 30s
        trip: consecutiveFailures >= 5

  targets:
    apps:
      checkout-service:
        retry: standard-retry
        timeout: standard-timeout
        circuitBreaker: standard-cb

    components:
      statestore:
        outbound:
          retry: standard-retry
          timeout: standard-timeout
```

## Retry Policies

Dapr supports three retry policy types:

```yaml
retries:
  # Constant interval retry
  constant-retry:
    policy: constant
    duration: 2s
    maxRetries: 5

  # Exponential backoff retry
  exponential-retry:
    policy: exponential
    initialInterval: 100ms
    randomizationFactor: 0.5
    multiplier: 2
    maxInterval: 30s
    maxElapsedTime: 5m
    maxRetries: -1    # Unlimited, bounded by maxElapsedTime
```

## Timeout Configuration

Timeouts apply to individual requests:

```yaml
timeouts:
  fast-timeout: 1s
  standard-timeout: 10s
  slow-timeout: 60s
```

Assign different timeouts to different targets based on expected response times.

## Circuit Breaker Configuration

The circuit breaker trips after a threshold of failures:

```yaml
circuitBreakers:
  payment-cb:
    maxRequests: 2       # Max concurrent requests when half-open
    interval: 0s         # Cyclic period - 0 means never resets
    timeout: 30s         # How long before transitioning half-open
    trip: consecutiveFailures >= 3
```

The `trip` expression supports:
- `consecutiveFailures >= N`
- `failureRatio >= 0.5` (50% failure rate)

## Targeting Components

Apply resiliency policies to Dapr components:

```yaml
targets:
  components:
    pubsub:
      inbound:         # For subscriptions (incoming messages)
        retry: pub-retry
      outbound:        # For publish operations
        retry: standard-retry
        timeout: standard-timeout
    statestore:
      outbound:
        circuitBreaker: state-cb
        timeout: fast-timeout
```

## Scoping Resiliency Policies

Limit which apps use a resiliency policy:

```yaml
metadata:
  name: frontend-resiliency
scopes:
  - frontend-service
```

## Applying the Policy

Deploy the resiliency YAML to Kubernetes:

```bash
kubectl apply -f resiliency.yaml -n production
```

Verify it was loaded:

```bash
kubectl get resiliency -n production
dapr logs --app-id my-service -k | grep resiliency
```

## Summary

The Dapr resiliency specification uses the `Resiliency` kind to define named retry, timeout, and circuit breaker policies that are then assigned to app or component targets. Exponential backoff retries with randomization prevent thundering herd issues, while circuit breakers protect downstream services from cascading failures. Scoping ensures policies apply only to the intended applications.
