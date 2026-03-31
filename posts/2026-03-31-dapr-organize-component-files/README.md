# How to Organize Dapr Component Files in a Project

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Component, Project Structure, GitOps, DevOps

Description: Learn how to organize Dapr component YAML files in a project for clarity, reusability, and safe GitOps-based deployments across environments.

---

## Why Organization Matters

As your project grows, you may accumulate dozens of Dapr component files. Without a clear structure, it becomes hard to know which components belong to which environment, which services use them, and how to safely promote changes.

## Recommended Directory Structure

```text
infra/
  dapr/
    base/
      kustomization.yaml
      statestore.yaml
      pubsub.yaml
      secretstore.yaml
    overlays/
      staging/
        kustomization.yaml
        statestore-patch.yaml
      production/
        kustomization.yaml
        statestore-patch.yaml
    resiliency/
      default-resiliency.yaml
    configuration/
      appconfig.yaml
```

## Base Components

Base components contain the shared structure with placeholder or dev-safe values:

```yaml
# infra/dapr/base/statestore.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
    - name: redisHost
      value: "redis:6379"
    - name: redisPassword
      value: ""
```

## Overlay Patches for Environments

Kustomize patches override only what changes per environment:

```yaml
# infra/dapr/overlays/production/statestore-patch.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  metadata:
    - name: redisHost
      value: "redis-prod.cache.windows.net:6380"
    - name: redisPassword
      secretKeyRef:
        name: redis-prod-secret
        key: password
    - name: enableTLS
      value: "true"
```

```yaml
# infra/dapr/overlays/production/kustomization.yaml
resources:
  - ../../base
patches:
  - path: statestore-patch.yaml
```

## Applying with Kustomize

```bash
# Preview what will be applied
kubectl kustomize infra/dapr/overlays/production

# Apply to production namespace
kubectl apply -k infra/dapr/overlays/production -n production

# Apply to staging
kubectl apply -k infra/dapr/overlays/staging -n staging
```

## Naming Conventions

Consistent naming reduces confusion:

| File Name | Purpose |
|---|---|
| `statestore.yaml` | Primary state store component |
| `pubsub-kafka.yaml` | Kafka pub/sub component |
| `binding-s3-input.yaml` | S3 input binding |
| `default-resiliency.yaml` | Cluster-wide resiliency policy |

## Grouping by Building Block

An alternative to env-based structure is grouping by Dapr building block:

```text
dapr/
  state/
  pubsub/
  bindings/
  secrets/
  configuration/
  resiliency/
```

## Summary

A well-organized Dapr component directory using base/overlay Kustomize patterns keeps your infrastructure DRY, environment-specific differences explicit, and GitOps promotions safe. Consistent naming conventions reduce cognitive overhead as the number of components grows.
