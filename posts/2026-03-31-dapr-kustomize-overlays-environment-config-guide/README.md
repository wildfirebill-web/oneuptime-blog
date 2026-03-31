# How to Use Kustomize Overlays for Dapr Environment Configuration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kustomize, Kubernetes, Environment Configuration, GitOps

Description: Learn how to manage Dapr component configurations across dev, staging, and production environments using Kustomize overlays with patches and strategic merge strategies.

---

## Why Kustomize for Dapr Configuration

Dapr components are Kubernetes resources. Managing them across multiple environments - dev, staging, production - without duplicating YAML files is a common challenge. Kustomize solves this by letting you define a base configuration and apply environment-specific patches on top, keeping the diff between environments small and explicit.

## Directory Structure

```text
/k8s
  /base
    kustomization.yaml
    statestore.yaml
    pubsub.yaml
    secrets.yaml
    subscription.yaml
    dapr-config.yaml
  /overlays
    /dev
      kustomization.yaml
      statestore-patch.yaml
    /staging
      kustomization.yaml
      statestore-patch.yaml
      pubsub-patch.yaml
    /production
      kustomization.yaml
      statestore-patch.yaml
      pubsub-patch.yaml
      secrets-patch.yaml
```

## Base Kustomization

```yaml
# k8s/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - statestore.yaml
  - pubsub.yaml
  - secrets.yaml
  - subscription.yaml
  - dapr-config.yaml
commonLabels:
  app.kubernetes.io/managed-by: kustomize
```

## Base Component Files

```yaml
# k8s/base/statestore.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: default
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis:6379"
  - name: redisPassword
    value: ""
  - name: actorStateStore
    value: "true"
  - name: enableTLS
    value: "false"
```

```yaml
# k8s/base/pubsub.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: messagebus
  namespace: default
spec:
  type: pubsub.rabbitmq
  version: v1
  metadata:
  - name: host
    value: "amqp://rabbitmq:5672"
  - name: durable
    value: "true"
  - name: prefetchCount
    value: "10"
```

## Dev Overlay

```yaml
# k8s/overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
namespace: dev
patches:
  - path: statestore-patch.yaml
    target:
      kind: Component
      name: statestore
commonLabels:
  environment: dev
```

```yaml
# k8s/overlays/dev/statestore-patch.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  metadata:
  - name: redisHost
    value: "localhost:6379"
  - name: redisPassword
    value: ""
```

## Production Overlay

```yaml
# k8s/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
namespace: production
patches:
  - path: statestore-patch.yaml
    target:
      kind: Component
      name: statestore
  - path: pubsub-patch.yaml
    target:
      kind: Component
      name: messagebus
  - path: secrets-patch.yaml
    target:
      kind: Component
      name: statestore
commonLabels:
  environment: production
```

```yaml
# k8s/overlays/production/statestore-patch.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  metadata:
  - name: redisHost
    value: "redis-cluster.production.svc.cluster.local:6379"
  - name: enableTLS
    value: "true"
  - name: redisPassword
    secretKeyRef:
      name: redis-credentials
      key: password
```

```yaml
# k8s/overlays/production/pubsub-patch.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: messagebus
spec:
  metadata:
  - name: host
    value: "amqp://rabbitmq-ha.production.svc.cluster.local:5672"
  - name: prefetchCount
    value: "50"
  - name: publisherConfirm
    value: "true"
```

## Using JSON Patches for Fine-Grained Changes

For patching specific array elements, use a JSON patch:

```yaml
# k8s/overlays/staging/add-resiliency-patch.yaml
- op: add
  path: /spec/metadata/-
  value:
    name: maxRetries
    value: "5"
- op: replace
  path: /spec/metadata/0/value
  value: "redis-staging.svc.cluster.local:6379"
```

```yaml
# k8s/overlays/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
namespace: staging
patches:
  - target:
      kind: Component
      name: statestore
    path: add-resiliency-patch.yaml
```

## Applying Overlays

```bash
# Preview what will be applied
kubectl kustomize k8s/overlays/dev

# Apply dev environment
kubectl apply -k k8s/overlays/dev

# Apply staging
kubectl apply -k k8s/overlays/staging

# Apply production
kubectl apply -k k8s/overlays/production
```

## Validating Before Apply

Use `--dry-run` to validate without applying:

```bash
kubectl apply -k k8s/overlays/production --dry-run=client
```

Or use the kustomize CLI directly:

```bash
# Install kustomize
brew install kustomize

# Build and validate
kustomize build k8s/overlays/production | kubectl apply --dry-run=client -f -
```

## GitOps Integration with ArgoCD

Reference the overlay in an ArgoCD Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dapr-components-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myrepo
    targetRevision: main
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Summary

Kustomize overlays provide a clean, DRY approach to managing Dapr component configurations across environments. Define your base components once, then apply environment-specific patches that only change what differs - connection strings, TLS settings, replica counts, and secret references. The overlay structure integrates naturally with GitOps tools like ArgoCD, enabling fully automated, auditable configuration promotion from dev through production.
