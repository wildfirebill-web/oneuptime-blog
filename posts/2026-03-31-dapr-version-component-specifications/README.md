# How to Version Dapr Component Specifications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Component, Versioning, YAML, Upgrade

Description: Learn how to version Dapr component specifications to manage upgrades, maintain backward compatibility, and safely migrate to new component versions.

---

## Why Component Versioning Matters

Dapr component specs include a `version` field that maps to the component implementation version. When a new version of a component adds breaking changes or new required fields, versioning lets you upgrade at your own pace without disrupting running services.

## The Version Field

Every Dapr component has a `spec.version` field:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: mystore
spec:
  type: state.redis
  version: v1      # component implementation version
  metadata:
    - name: redisHost
      value: "localhost:6379"
```

Dapr maintains a registry of which `version` values are valid per `type`. The common pattern is `v1`, with `v2` or later used when breaking changes are introduced.

## Listing Available Versions

```bash
# Run Dapr locally and check the component registry logs
dapr run --app-id probe --log-level debug -- sleep 2 2>&1 | grep "component registered"

# For a Kubernetes cluster, describe the component to see the status
kubectl describe component mystore -n production
```

## Running Multiple Versions Side by Side

During a migration, you can run both versions simultaneously under different names:

```yaml
# v1 component - existing services
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore-v1
spec:
  type: state.postgresql
  version: v1
  metadata:
    - name: connectionString
      secretKeyRef:
        name: pg-secret
        key: connstr
---
# v2 component - new services
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore-v2
spec:
  type: state.postgresql
  version: v2
  metadata:
    - name: connectionString
      secretKeyRef:
        name: pg-secret
        key: connstr
    - name: tablePrefix
      value: "dapr_"
```

Scope each version to the appropriate app IDs:

```yaml
scopes:
  - legacy-order-service   # uses statestore-v1
```

## Migrating from v1 to v2

```bash
# Step 1: deploy the new component alongside the old one
kubectl apply -f statestore-v2.yaml -n production

# Step 2: update the new service deployments to reference statestore-v2
# Step 3: verify the new service is healthy
kubectl rollout status deployment/new-order-service -n production

# Step 4: remove the v1 component once all services have migrated
kubectl delete component statestore-v1 -n production
```

## Pinning Versions in GitOps

In a GitOps workflow, keep component version changes in a separate commit so they are easy to revert:

```bash
git diff HEAD~1 -- components/
# Expect only version field changes in a version-bump commit
```

## Summary

Dapr component versioning gives you a controlled migration path when component implementations change. Running multiple versions side by side with scope-based routing ensures zero-downtime upgrades while you incrementally move services to the newer implementation.
