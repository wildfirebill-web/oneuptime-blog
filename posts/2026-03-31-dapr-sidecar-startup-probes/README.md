# How to Configure Dapr Sidecar Startup Probes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Sidecar, Kubernetes, Probe, Configuration

Description: Configure Kubernetes startup probes for the Dapr sidecar to prevent premature readiness checks and avoid pod restart loops during slow component initialization.

---

Kubernetes startup probes let you tell the kubelet to wait until a container is fully initialized before activating liveness and readiness probes. This is especially useful for the Dapr sidecar when it connects to slow backends like Cosmos DB or Kafka during startup.

## Why Startup Probes Matter for Dapr

Without a startup probe, Kubernetes may run liveness checks before daprd has finished loading all components. If a component like a state store takes more than a few seconds to connect, the liveness probe can fail, causing the pod to restart in a loop.

A startup probe gives daprd the time it needs to initialize before the kubelet starts applying liveness checks.

## Default Dapr Probe Configuration

The Dapr injector automatically configures readiness and liveness probes for the sidecar container. You can override them using pod annotations.

```yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "order-service"
  dapr.io/sidecar-startup-probe-threshold: "6"
  dapr.io/sidecar-startup-probe-period-seconds: "10"
  dapr.io/sidecar-startup-probe-failure-threshold: "3"
```

These values configure:
- `threshold`: number of seconds after the container starts before the probe is initiated
- `period-seconds`: how often to run the probe
- `failure-threshold`: how many failures before the container is marked as failed

## Manually Inspecting the Injected Probe

After deploying, inspect the pod spec to see the injected probe:

```bash
kubectl get pod my-pod -o json | jq '.spec.containers[] | select(.name=="daprd") | .startupProbe'
```

Example output:

```json
{
  "httpGet": {
    "path": "/v1.0/healthz",
    "port": 3500
  },
  "failureThreshold": 3,
  "periodSeconds": 10
}
```

## Tuning for Slow Component Backends

If your state store or pub/sub broker takes a long time to be reachable at startup (for example, after a cluster restart), increase the failure threshold:

```yaml
annotations:
  dapr.io/sidecar-startup-probe-failure-threshold: "10"
  dapr.io/sidecar-startup-probe-period-seconds: "15"
```

This allows up to 150 seconds (10 x 15) for daprd to become healthy.

## Testing Probe Behavior

Simulate a slow startup by temporarily pointing a component at an unreachable endpoint and observing how Kubernetes handles it:

```bash
kubectl describe pod my-pod | grep -A 10 "Events:"
```

You should see `Startup probe failed` events if the probe is misconfigured or the backend is unreachable.

## Combining with App Startup

If your application also has a slow startup, configure its probe separately. Dapr waits for both the sidecar and the app to be healthy before routing traffic.

```yaml
annotations:
  dapr.io/sidecar-startup-probe-failure-threshold: "10"
  dapr.io/app-startup-probe-failure-threshold: "20"
```

## Summary

Configuring startup probes for the Dapr sidecar prevents premature liveness checks that cause unnecessary pod restarts. By tuning the threshold and period based on how long your components take to initialize, you ensure stable deployments even when connecting to slow or remote backends.
