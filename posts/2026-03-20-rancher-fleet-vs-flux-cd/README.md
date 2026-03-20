# Rancher Fleet vs Flux CD: GitOps Comparison

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: fleet, flux-cd, gitops, kubernetes, comparison

Description: A comprehensive comparison of Rancher Fleet and Flux CD for GitOps-based Kubernetes management, focusing on multi-cluster support, tooling, and operational experience.

## Overview

Flux CD and Rancher Fleet are both popular GitOps tools for Kubernetes, but they approach the problem from different angles. Flux is a CNCF-graduated project with a modular toolkit and strong integration with the Kubernetes ecosystem. Fleet is SUSE Rancher's purpose-built GitOps engine optimized for large-scale multi-cluster management. This guide helps you evaluate which fits your needs.

## What Is Flux CD?

Flux CD is a CNCF-graduated GitOps tool consisting of multiple controllers (Source Controller, Kustomize Controller, Helm Controller, Notification Controller, and Image Automation Controller). Each controller is independent and composable. Flux integrates with Helm, Kustomize, and OCI registries, and is widely used across the Kubernetes ecosystem.

## What Is Rancher Fleet?

Fleet is SUSE Rancher's GitOps engine, designed specifically for managing applications across large numbers of clusters from a central control plane. It uses a bundle model and integrates natively with Rancher for authentication, RBAC, and cluster targeting.

## Feature Comparison

| Feature | Fleet | Flux CD |
|---|---|---|
| CNCF Status | Not a CNCF project | Graduated |
| Multi-cluster Scale | Thousands of clusters | Hundreds of clusters |
| Helm Support | Yes | Yes (Helm Controller) |
| Kustomize Support | Yes | Yes (Kustomize Controller) |
| OCI Registry Support | Yes | Yes |
| Image Automation | No | Yes (Image Automation Controller) |
| Notification Support | Limited | Yes (Notification Controller) |
| UI | Via Rancher | Weave GitOps (separate) |
| RBAC | Via Rancher | Kubernetes native |
| Drift Detection | Yes | Yes |
| Multi-tenancy | Via Rancher | Yes (tenants) |
| Air-gap Support | Yes | Yes |
| CLI | fleet CLI | flux CLI |
| Edge Support | Excellent | Limited |

## Flux Toolkit Architecture

```
┌─────────────────────────────────┐
│         Flux Controllers         │
│  ┌──────────────────────────┐   │
│  │  Source Controller        │   │ ← Watches Git, Helm, OCI
│  │  Kustomize Controller     │   │ ← Applies Kustomize overlays
│  │  Helm Controller          │   │ ← Manages Helm releases
│  │  Notification Controller  │   │ ← Sends alerts
│  │  Image Automation         │   │ ← Updates image tags in Git
│  └──────────────────────────┘   │
└─────────────────────────────────┘
```

## Defining GitOps Resources

### Fleet GitRepo

```yaml
# Fleet: sync a Git repo to clusters labeled env=staging
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: webapp
  namespace: fleet-default
spec:
  repo: https://github.com/myorg/webapp-config
  branch: main
  targets:
    - name: staging
      clusterSelector:
        matchLabels:
          env: staging
```

### Flux GitRepository + Kustomization

```yaml
# Flux Step 1: Define the Git source
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: webapp
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/webapp-config
  ref:
    branch: main
---
# Flux Step 2: Apply the Kustomization
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: webapp
  namespace: flux-system
spec:
  interval: 5m
  path: ./staging
  prune: true
  sourceRef:
    kind: GitRepository
    name: webapp
  targetNamespace: webapp
```

## Helm Management

### Fleet with Helm

```yaml
# fleet.yaml in chart directory
helm:
  repo: https://charts.myorg.com
  chart: webapp
  version: 1.2.3
  values:
    replicaCount: 2
    image:
      tag: v1.2.3
```

### Flux Helm Controller

```yaml
# Flux HelmRelease
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: webapp
  namespace: webapp
spec:
  interval: 5m
  chart:
    spec:
      chart: webapp
      version: '>=1.2.0 <2.0.0'
      sourceRef:
        kind: HelmRepository
        name: myorg-charts
  values:
    replicaCount: 2
```

## Image Automation

Flux has a unique feature that Fleet lacks: Image Automation. Flux can monitor container registries for new image tags and automatically update Git with the new image references:

```yaml
# Flux ImagePolicy - select latest semver image
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: webapp
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: webapp-repo
  policy:
    semver:
      range: '>=1.0.0'
```

## Multi-cluster with Flux

Flux multi-cluster management requires bootstrapping Flux on each cluster individually and using a central management repository to configure each cluster's Kustomizations. This works well but requires more planning.

Fleet's multi-cluster management is centralized by design — a single Fleet controller in Rancher manages all clusters.

## When to Choose Fleet

- You run Rancher and want tight integration
- Large-scale edge cluster management (100+ clusters)
- Centralized cluster targeting with label-based routing
- Simplified operations with single control plane

## When to Choose Flux CD

- CNCF-graduation and community support are important factors
- Image automation (auto-updating image tags in Git) is needed
- Modular, composable tooling is preferred
- You use Weave GitOps or other Flux-compatible UIs
- Notification to Slack, Teams, or PagerDuty is required

## Conclusion

Flux CD and Fleet are both mature, capable GitOps tools. Flux wins on modularity, CNCF backing, and ecosystem integrations (especially image automation). Fleet wins on scale, simplicity in Rancher environments, and edge cluster management. Teams using Rancher should default to Fleet for cluster configuration management. Teams looking for a standalone CNCF-backed GitOps solution should evaluate Flux.
