# How to Use the dapr status Command for Control Plane Health

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, CLI, Health Check, Kubernetes, Control Plane

Description: Learn how to use the dapr status command to check the health of the Dapr control plane components running on a Kubernetes cluster.

---

## Overview

The `dapr status` command shows the health and version of each Dapr control plane component in a Kubernetes cluster. It reports whether the operator, sentry, placement, sidecar injector, dashboard, and scheduler are running correctly and exposes version information for each.

## Basic Usage

```bash
dapr status --kubernetes
```

Sample output:

```
  NAME                   NAMESPACE    HEALTHY  STATUS    REPLICAS  VERSION  AGE
  dapr-dashboard         dapr-system  True     Running   1         0.14.0   2h
  dapr-operator          dapr-system  True     Running   1         1.13.0   2h
  dapr-placement-server  dapr-system  True     Running   1         1.13.0   2h
  dapr-sentry            dapr-system  True     Running   1         1.13.0   2h
  dapr-sidecar-injector  dapr-system  True     Running   1         1.13.0   2h
  dapr-scheduler-server  dapr-system  True     Running   1         1.13.0   2h
```

## Checking a Custom Namespace

If you installed Dapr into a non-default namespace:

```bash
dapr status --kubernetes --namespace my-dapr-system
```

## JSON Output

```bash
dapr status --kubernetes --output json
```

Sample JSON output:

```json
[
  {
    "name": "dapr-operator",
    "namespace": "dapr-system",
    "healthy": "True",
    "status": "Running",
    "replicas": 1,
    "version": "1.13.0",
    "age": "2h"
  },
  {
    "name": "dapr-sentry",
    "namespace": "dapr-system",
    "healthy": "True",
    "status": "Running",
    "replicas": 1,
    "version": "1.13.0",
    "age": "2h"
  }
]
```

## Using in a Deployment Pipeline

Check Dapr control plane health before deploying workloads:

```bash
#!/bin/bash
echo "Checking Dapr control plane health..."
STATUS=$(dapr status --kubernetes --output json)

UNHEALTHY=$(echo $STATUS | jq '[.[] | select(.healthy != "True")] | length')

if [ "$UNHEALTHY" -gt 0 ]; then
  echo "ERROR: $UNHEALTHY Dapr control plane component(s) are unhealthy"
  echo $STATUS | jq '.[] | select(.healthy != "True")'
  exit 1
fi

echo "All Dapr control plane components are healthy"
```

## Diagnosing an Unhealthy Component

If a component shows `False` for health:

```bash
# Check pod events
kubectl describe pod -l app=dapr-sentry -n dapr-system

# Check pod logs
kubectl logs -l app=dapr-sentry -n dapr-system
```

## After an Upgrade

Run `dapr status` immediately after `dapr upgrade` to confirm all components updated successfully:

```bash
dapr upgrade --kubernetes --runtime-version 1.14.0
dapr status --kubernetes
```

All components should show the new version and `Running` status.

## Summary

`dapr status` is the go-to command for verifying that the Dapr control plane is healthy after installation, upgrades, or cluster restarts. Its JSON output integrates cleanly into CI/CD pipelines and monitoring scripts, enabling automated readiness checks before workload deployments proceed.
