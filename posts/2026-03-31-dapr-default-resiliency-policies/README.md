# How to Set Default Resiliency Policies in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Resiliency, Default Policy, Fault Tolerance, Configuration

Description: Learn how to define default resiliency policies in Dapr that apply globally to all services and components as a safety net when no specific policy is configured.

---

## Overview

Dapr Resiliency resources support a default target that acts as a catch-all, applying a policy to any service or component that does not have an explicit policy configured. This is a safety net that ensures baseline fault tolerance across your entire application without having to enumerate every service.

## Defining Default Policies

Use the special `default` key under `targets.apps` and `targets.components`:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: global-resiliency
  namespace: default
spec:
  policies:
    timeouts:
      defaultTimeout: 10s
      strictTimeout: 3s
    retries:
      defaultRetry:
        policy: exponential
        initialInterval: 500ms
        multiplier: 2.0
        maxInterval: 30s
        maxRetries: 5
        randomizationFactor: 0.5
    circuitBreakers:
      defaultCB:
        maxRequests: 1
        interval: 30s
        timeout: 60s
        trip: consecutiveFailures >= 5
  targets:
    apps:
      default:
        timeout: defaultTimeout
        retry: defaultRetry
        circuitBreaker: defaultCB
    components:
      default:
        outbound:
          timeout: defaultTimeout
          retry: defaultRetry
```

With this configuration, every Dapr service invocation and every component operation that does not have a more specific policy will use these defaults.

## Overriding Defaults for Specific Services

Specific targets override defaults. The most specific match wins:

```yaml
targets:
  apps:
    default:
      timeout: defaultTimeout
      retry: defaultRetry
    payment-service:
      timeout: strictTimeout
      retry: defaultRetry
      circuitBreaker: defaultCB
```

Here `payment-service` uses a 3-second timeout instead of the 10-second default.

## Default Policies for Components

Apply separate defaults to inbound (subscription delivery) and outbound (state/publish calls):

```yaml
targets:
  components:
    default:
      outbound:
        timeout: defaultTimeout
        retry: defaultRetry
      inbound:
        timeout: defaultTimeout
        retry: defaultRetry
```

## Namespace-Wide Defaults

A single `Resiliency` resource with defaults in a namespace provides a baseline for all services in that namespace. Deploy one per namespace:

```bash
kubectl apply -f default-resiliency.yaml -n production
kubectl apply -f default-resiliency.yaml -n staging
```

Services with no specific resiliency configuration automatically inherit the namespace defaults.

## Verifying Default Policy Application

Check what policies are applied to a running service:

```bash
kubectl get resiliency global-resiliency -o yaml
```

View sidecar logs to see default policies being evaluated:

```bash
kubectl logs deployment/any-service -c daprd \
  | grep -i "resiliency\|policy"
```

## Priority Order

When multiple `Resiliency` resources exist, Dapr applies policies in this priority order:
1. Exact match on the target app/component name in any `Resiliency` resource
2. Default target in any `Resiliency` resource
3. No policy (bare metal behavior)

## Summary

Default Resiliency policies provide a namespace-wide safety net by applying baseline timeout, retry, and circuit breaker settings to all services and components that lack explicit policies. This eliminates gaps in your resiliency coverage and ensures that even services added later automatically inherit sensible fault-tolerance behavior.
