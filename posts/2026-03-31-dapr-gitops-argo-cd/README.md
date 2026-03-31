# How to Implement GitOps for Dapr with Argo CD

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Argo CD, GitOps, Kubernetes, DevOps

Description: Learn how to implement GitOps for Dapr applications using Argo CD to automatically sync Dapr components, configurations, and application manifests from Git.

---

GitOps with Argo CD ensures your Dapr components and application configuration are always in sync with Git. Argo CD continuously reconciles the cluster state against the desired state in your Git repository, enabling auditable, automated deployments.

## Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for Argo CD to be ready
kubectl wait --for=condition=available deployment -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=5m

# Get the initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d
```

## Repository Structure for GitOps

```bash
dapr-gitops/
├── apps/
│   └── order-service/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── kustomization.yaml
├── dapr-components/
│   ├── pubsub.yaml
│   ├── state-store.yaml
│   └── secret-store.yaml
├── dapr-config/
│   └── configuration.yaml
└── argocd/
    ├── app-of-apps.yaml
    ├── dapr-components-app.yaml
    └── order-service-app.yaml
```

## Argo CD Application for Dapr Components

```yaml
# argocd/dapr-components-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dapr-components
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/dapr-gitops.git
    targetRevision: HEAD
    path: dapr-components
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

## Argo CD Application for the Dapr App

```yaml
# argocd/order-service-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/dapr-gitops.git
    targetRevision: HEAD
    path: apps/order-service
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

## App-of-Apps Pattern

```yaml
# argocd/app-of-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dapr-platform
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/dapr-gitops.git
    targetRevision: HEAD
    path: argocd
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Deploying and Managing with Argo CD CLI

```bash
# Install Argo CD CLI
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && mv argocd /usr/local/bin/

# Login
argocd login localhost:8080 --username admin --password <password>

# Apply the app-of-apps
kubectl apply -f argocd/app-of-apps.yaml

# Check sync status
argocd app list
argocd app get dapr-components

# Manually sync if needed
argocd app sync dapr-components --prune
```

## Summary

Argo CD implements GitOps for Dapr by continuously reconciling cluster state with Git. Use the App-of-Apps pattern to manage Dapr components, configurations, and applications as separate Argo CD applications with independent sync policies. Enable `automated` sync with `selfHeal` to automatically fix manual changes, and `prune` to remove resources deleted from Git.
