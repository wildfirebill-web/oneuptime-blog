# How to Configure Dapr Operator on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kubernetes, Operator, Control Plane, Configuration

Description: Configure the Dapr Operator on Kubernetes to manage component CRD reconciliation, set resource limits, enable HA mode, and tune operator behavior.

---

## What Is the Dapr Operator?

The Dapr Operator is a Kubernetes controller that watches Dapr component CRDs (Components, Configurations, Resiliency, Subscriptions) and distributes their configuration to running Dapr sidecars. It is a critical piece of the Dapr control plane.

## Viewing Current Operator Configuration

```bash
# Check operator pod status
kubectl get pods -n dapr-system -l app=dapr-operator

# View operator logs
kubectl logs -n dapr-system -l app=dapr-operator --tail=50

# Check operator deployment spec
kubectl describe deployment dapr-operator -n dapr-system
```

## Configuring the Operator via Helm

```yaml
# dapr-operator-values.yaml
dapr_operator:
  replicaCount: 2
  logLevel: info
  watchdogEnabled: true
  watchdogInterval: 500ms
  watchdogMaxRestartsPerMin: 3
  resources:
    requests:
      cpu: "100m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
  nodeSelector:
    kubernetes.io/os: linux
  tolerations: []
  affinity: {}
```

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  -f dapr-operator-values.yaml
```

## Understanding the Watchdog Feature

The Dapr Operator watchdog monitors running Dapr sidecars and can restart those that become unresponsive:

```bash
# Enable the watchdog in the operator
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --set dapr_operator.watchdogEnabled=true \
  --set dapr_operator.watchdogInterval=500ms \
  --set dapr_operator.watchdogMaxRestartsPerMin=3
```

## Checking Component Reconciliation

```bash
# List all Dapr components the operator is managing
kubectl get components.dapr.io -A

# Verify a component was picked up by the operator
kubectl describe component statestore -n default

# Watch for component update events
kubectl get events -n default --field-selector reason=ComponentUpdated
```

## Configuring Operator RBAC

The Dapr Operator needs ClusterRole permissions to watch all namespaces. Verify it has the correct permissions:

```bash
# Check the operator's ClusterRoleBinding
kubectl get clusterrolebinding dapr-operator

# View the operator's service account permissions
kubectl describe clusterrole dapr-operator
```

## Restarting the Operator

When components are not being reconciled, restart the operator:

```bash
kubectl rollout restart deployment/dapr-operator -n dapr-system
kubectl rollout status deployment/dapr-operator -n dapr-system
```

## Summary

The Dapr Operator manages CRD reconciliation and keeps sidecar configurations synchronized with the cluster state. Configure it via Helm values to set replica count for HA, resource limits, and enable the watchdog feature. Monitoring operator logs helps diagnose issues where component changes are not propagating to running sidecars.
