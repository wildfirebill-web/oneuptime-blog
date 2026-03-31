# How to Use Dapr with Argo CD for GitOps

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Argo CD, GitOps, Kubernetes, Deployment

Description: Manage Dapr microservice deployments using Argo CD's GitOps model, syncing Dapr component configurations and application manifests from Git to Kubernetes.

---

## Overview

Argo CD is a declarative GitOps continuous delivery tool for Kubernetes. Combining Argo CD with Dapr provides a powerful pattern where all Dapr component configurations, subscriptions, and application deployments are version-controlled in Git and automatically synced to your cluster.

## Prerequisites

- Argo CD installed on Kubernetes
- A Git repository with Dapr manifests
- Dapr installed on the target cluster

## Repository Structure

Organize your Git repository for GitOps:

```
dapr-gitops/
  apps/
    order-service/
      deployment.yaml
      service.yaml
  components/
    statestore.yaml
    pubsub.yaml
  subscriptions/
    orders-sub.yaml
  argocd/
    app-order-service.yaml
    app-components.yaml
```

## Argo CD Application for Dapr Components

Create an Argo CD Application that manages Dapr components:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dapr-components
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/dapr-gitops
    targetRevision: main
    path: components
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

## Argo CD Application for a Dapr Service

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/dapr-gitops
    targetRevision: main
    path: apps/order-service
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
```

## Managing Dapr Component Secrets with Argo CD

Use Argo CD's integration with Vault or Sealed Secrets to manage component credentials:

```bash
# Install Sealed Secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  -n kube-system

# Seal a Dapr component secret
kubectl create secret generic redis-credentials \
  --from-literal=password=s3cr3t \
  --dry-run=client -o json | \
  kubeseal --controller-namespace kube-system \
  -o yaml > sealed-redis-credentials.yaml
```

## Argo CD App of Apps Pattern

For managing many Dapr services, use the App of Apps pattern:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dapr-platform
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/dapr-gitops
    targetRevision: main
    path: argocd
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Health Checks for Dapr Applications

```yaml
# Custom health check for Dapr-annotated deployments
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  # ...
  ignoreDifferences:
  - group: dapr.io
    kind: Component
    jsonPointers:
    - /metadata/annotations/kubectl.kubernetes.io~1last-applied-configuration
```

## Rolling Out Dapr Component Changes

When changing Dapr component versions or configuration, Argo CD handles the rollout:

```bash
# Check sync status
argocd app get dapr-components

# Manually trigger sync with replacement
argocd app sync dapr-components --force --replace

# Monitor rollout
argocd app wait dapr-components --health --timeout 120
```

## Summary

Argo CD's GitOps model is an excellent fit for Dapr microservice deployments. By storing Dapr component configurations, subscriptions, and application manifests in Git, you gain a single source of truth, automated drift detection, and auditable deployment history. The App of Apps pattern scales this approach across large Dapr-powered platforms.
