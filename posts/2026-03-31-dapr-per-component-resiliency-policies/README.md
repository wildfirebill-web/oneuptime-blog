# How to Configure Per-Component Resiliency Policies in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Resiliency, Component, Policy, Configuration

Description: Learn how to configure per-component resiliency policies in Dapr to apply independent retry, timeout, and circuit breaker settings to individual components.

---

## Overview

Dapr's resiliency framework lets you assign distinct policies to each component rather than applying global settings. This means your Redis state store can have aggressive retries while your payment API uses a strict circuit breaker - each tuned to its own failure characteristics.

Per-component policies are defined in the `targets.components` section of a Resiliency resource.

## Resiliency Policy Structure

A full resiliency resource with per-component targets looks like this:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: app-resiliency
  namespace: default
spec:
  policies:
    timeouts:
      shortTimeout: 2s
      longTimeout: 10s
    retries:
      fastRetry:
        policy: constant
        duration: 500ms
        maxRetries: 3
      slowRetry:
        policy: exponential
        maxRetries: 5
        maxInterval: 30s
    circuitBreakers:
      strictCB:
        maxRequests: 1
        timeout: 60s
        trip: consecutiveFailures >= 2
      looseCB:
        maxRequests: 3
        timeout: 15s
        trip: consecutiveFailures >= 5
  targets:
    components:
      redis-statestore:
        outbound:
          timeout: shortTimeout
          retry: fastRetry
      payment-pubsub:
        outbound:
          timeout: longTimeout
          retry: slowRetry
          circuitBreaker: strictCB
      mysql-binding:
        outbound:
          retry: slowRetry
          circuitBreaker: looseCB
```

## Applying Different Policies to State Stores

State stores typically need fast retries because Redis operations are usually milliseconds:

```yaml
targets:
  components:
    redis-statestore:
      outbound:
        timeout: 1s
        retry: fastRetry
```

Test that this policy activates correctly:

```bash
curl -X POST http://localhost:3500/v1.0/state/redis-statestore \
  -H "Content-Type: application/json" \
  -d '[{"key": "test-key", "value": "hello"}]'
```

## Applying a Circuit Breaker to a Message Bus

Pub/sub components benefit from circuit breakers to avoid flooding a degraded broker:

```yaml
targets:
  components:
    rabbitmq-pubsub:
      outbound:
        circuitBreaker: strictCB
        retry: slowRetry
```

## Scoping Policies to Specific Namespaces

Resiliency resources are namespace-scoped. Deploy separate resources per namespace to avoid cross-namespace policy bleed:

```bash
kubectl apply -f resiliency-production.yaml -n production
kubectl apply -f resiliency-staging.yaml -n staging
```

## Verifying Policy Assignment

Check that policies are loaded by inspecting Dapr sidecar logs:

```bash
kubectl logs -l app=my-service -c daprd | grep "resiliency"
```

Expected output:

```text
level=info msg="Found resiliency policy for component redis-statestore"
level=info msg="Found resiliency policy for component payment-pubsub"
```

## Combining App and Component Policies

You can mix app-level and component-level targets in the same resource:

```yaml
targets:
  apps:
    inventory-service:
      circuitBreaker: looseCB
  components:
    mongo-statestore:
      outbound:
        retry: slowRetry
```

This gives fine-grained control while keeping the policy definitions centralized.

## Summary

Per-component resiliency policies in Dapr allow you to tailor retry, timeout, and circuit breaker settings independently for each infrastructure component. Defining separate policies for state stores, pub/sub brokers, and bindings ensures each integration point handles failures according to its own tolerance profile, improving overall system reliability.
