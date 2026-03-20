# Rancher Fleet vs ArgoCD: GitOps Comparison

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Fleet, ArgoCD, GitOps, Kubernetes, Comparison

Description: A detailed comparison of Rancher Fleet and ArgoCD for GitOps-based Kubernetes deployments, covering features, scalability, and multi-cluster support.

## Overview

GitOps has become the standard approach for managing Kubernetes workloads at scale. Rancher Fleet and ArgoCD are two leading GitOps tools, each with distinct architectures and strengths. Fleet is optimized for managing thousands of clusters from a single Git repository, while ArgoCD excels at application delivery with a rich UI and powerful sync strategies. This guide compares them to help you choose the right tool.

## What Is Rancher Fleet?

Fleet is a GitOps tool built by SUSE Rancher, designed from the ground up for managing large numbers of Kubernetes clusters. It uses a bundle-based model where Git repositories are continuously synced to target clusters. Fleet is tightly integrated with Rancher and is particularly strong in edge and multi-cluster scenarios.

## What Is ArgoCD?

ArgoCD is a declarative GitOps continuous delivery tool for Kubernetes. It monitors Git repositories for changes and automatically applies them to target clusters. ArgoCD features a rich web UI, multi-cluster support, sync waves, health assessments, and a powerful Application API.

## Feature Comparison

| Feature | Fleet | ArgoCD |
|---|---|---|
| Multi-cluster Scale | Thousands of clusters | Hundreds of clusters |
| Web UI | Basic | Rich / Advanced |
| Sync Strategies | Push-based | Push and Pull |
| Helm Support | Yes | Yes |
| Kustomize Support | Yes | Yes |
| Raw YAML Support | Yes | Yes |
| SSO Integration | Via Rancher | Yes (Dex built-in) |
| RBAC | Via Rancher | Built-in |
| Notifications | Limited | Yes (Notifications controller) |
| Application Health | Basic | Advanced |
| Sync Waves | No | Yes |
| Rollback | Yes (git-based) | Yes |
| App of Apps Pattern | Yes (bundles) | Yes |
| Edge Support | Excellent | Limited |
| Air-gap Support | Yes | Yes |
| Drift Detection | Yes | Yes |
| Progressive Delivery | No (use Flagger) | No (use Argo Rollouts) |

## Architecture

### Fleet Architecture

```text
Git Repository
      |
      v
Fleet Manager (Rancher)
      |
   +--+--+
   |     |
Fleet   Fleet
Agent   Agent
(Cluster 1)  (Cluster 2..N)
```

Fleet uses a bundle system. Each bundle maps to a directory in Git and can target clusters based on labels, cluster groups, or namespaces.

### ArgoCD Architecture

```text
Git Repository
      |
      v
ArgoCD Application Controller
      |
ArgoCD API Server + UI
      |
   +--+--+
   |     |
Cluster  Cluster
1        2
```

ArgoCD manages applications as Kubernetes Custom Resources (Application CRD). Each Application maps a Git source to a destination cluster/namespace.

## Defining GitOps Resources

### Fleet GitRepo

```yaml
# Fleet GitRepo resource - targets clusters with label env=production

apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: my-app
  namespace: fleet-default
spec:
  repo: https://github.com/myorg/my-app-config
  branch: main
  targets:
    - name: production
      clusterSelector:
        matchLabels:
          env: production
  paths:
    - manifests/
```

### ArgoCD Application

```yaml
# ArgoCD Application resource
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/my-app-config
    targetRevision: HEAD
    path: manifests/
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Multi-cluster Scale

Fleet is the clear winner for large-scale multi-cluster deployments. It was designed to manage thousands of edge clusters efficiently, using a push model that scales horizontally. The Fleet agent on each cluster pulls updates from the Fleet controller.

ArgoCD can manage multiple clusters but was not designed for thousands of edge nodes. For very large deployments, Argo CD requires horizontal scaling of the application controller.

## UI and Observability

ArgoCD provides one of the best GitOps UIs available, with a real-time graph view of application resources, diff visualization, sync status per resource, and detailed event logs.

Fleet's UI is basic and integrated into the Rancher UI. For sophisticated observability, teams typically rely on Rancher monitoring and external tooling.

## Sync Waves and Ordering

ArgoCD supports sync waves via annotations, allowing precise ordering of resource creation:

```yaml
# Deploy database before application
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"   # Database deploys first
```

Fleet provides ordering through bundle dependencies rather than fine-grained sync waves.

## When to Choose Fleet

- You are managing dozens to thousands of clusters (especially edge)
- You are already using Rancher
- Simplicity and tight Rancher integration are priorities
- Edge and disconnected environments are targets

## When to Choose ArgoCD

- You need a rich UI and advanced observability
- Sync waves, hooks, and fine-grained sync control are important
- You use ArgoCD Notifications for deployment alerts
- Your GitOps scope is within a smaller number of clusters
- Your team is not using Rancher

## Conclusion

Fleet and ArgoCD are both excellent GitOps tools that excel in different scenarios. Fleet's standout strength is scale - managing thousands of clusters from a single control plane, especially in Rancher environments. ArgoCD's standout strength is depth - rich UI, sync waves, health assessment, and a mature application delivery API. Many organizations use ArgoCD for application delivery and Fleet for cluster configuration management, using the two tools complementarily.
