# How to Set Up GitOps with Rancher and Fleet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, GitOps, Fleet, CI/CD

Description: Implement GitOps workflows using Rancher Fleet to continuously deliver and synchronize Kubernetes workloads from Git repositories to multiple clusters.

## Introduction

Fleet is Rancher's built-in GitOps engine. It watches Git repositories and continuously reconciles the desired state in those repositories against the actual state in your Kubernetes clusters. Fleet scales from a single cluster to thousands, making it ideal for both simple single-cluster GitOps and complex multi-cluster fleet management.

## Prerequisites

- Rancher v2.6+ with Fleet enabled (it's enabled by default)
- A Git repository (GitHub, GitLab, Gitea, etc.)
- `kubectl` with access to the Rancher management cluster

## Understanding Fleet Concepts

| Concept | Description |
|---|---|
| **GitRepo** | A pointer to a Git repo + branch. Fleet watches this and syncs it. |
| **Bundle** | The compiled output of a GitRepo path — Helm charts, raw YAML, or Kustomize. |
| **ClusterGroup** | A logical grouping of clusters for targeting deployments. |
| **BundleDeployment** | A record of a Bundle deployed to a specific cluster. |

## Step 1: Organize Your Git Repository

Fleet supports multiple manifest formats in a single repo:

```
my-app-repo/
├── fleet.yaml                  # Root-level Fleet config (optional)
├── deploy/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── helm/
│   ├── fleet.yaml              # Helm chart config
│   └── values.yaml
└── kustomize/
    ├── kustomization.yaml
    ├── base/
    └── overlays/
        ├── dev/
        └── prod/
```

## Step 2: Create a Basic fleet.yaml

```yaml
# fleet.yaml — configures how Fleet handles this directory
defaultNamespace: production

# Target specific clusters
targets:
  - name: production-clusters
    clusterSelector:
      matchLabels:
        environment: production

  - name: dev-clusters
    clusterSelector:
      matchLabels:
        environment: development
    # Override values for dev clusters
    helm:
      values:
        replicaCount: 1
        resources:
          limits:
            memory: 256Mi
```

## Step 3: Create a GitRepo Resource

```yaml
# gitrepo-myapp.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: my-application
  namespace: fleet-default     # fleet-default = manage all imported clusters
spec:
  # Git repository URL
  repo: https://github.com/my-org/my-app-repo
  branch: main

  # Paths within the repo to sync (relative to repo root)
  paths:
    - deploy/

  # Poll interval (default: 15 seconds)
  pollingInterval: 30s

  # Optional: use a private repo with SSH or HTTP credentials
  # clientSecretName: github-credentials

  # Target clusters
  targets:
    - name: all-production
      clusterSelector:
        matchLabels:
          environment: production
```

```bash
kubectl apply -f gitrepo-myapp.yaml
kubectl get gitrepo -n fleet-default
```

## Step 4: Use Private Git Repositories

```bash
# Create credentials for a private GitHub repo (HTTPS)
kubectl create secret generic github-credentials \
  -n fleet-default \
  --from-literal=username=my-github-user \
  --from-literal=password=ghp_xxxxxxxxxxxx   # GitHub Personal Access Token

# For SSH:
kubectl create secret generic github-ssh \
  -n fleet-default \
  --from-file=ssh-privatekey=/path/to/id_ed25519 \
  --from-literal=known_hosts="github.com ssh-ed25519 AAAAC3..."
```

```yaml
# Reference the secret in the GitRepo
spec:
  clientSecretName: github-credentials
```

## Step 5: Deploy Helm Charts via Fleet

```yaml
# fleet.yaml inside a Helm chart directory
helm:
  chart: ./
  releaseName: my-app
  version: "1.2.0"
  values:
    image:
      tag: "v1.2.0"
  valuesFiles:
    - values.yaml
  # Per-cluster value overrides
  valuesFrom:
    - secretKeyRef:
        name: cluster-specific-values
        key: values.yaml
        namespace: fleet-system
```

## Step 6: Use Kustomize Overlays for Multi-Environment

```yaml
# kustomize/fleet.yaml
kustomize:
  dir: overlays/production

targets:
  - name: prod
    clusterSelector:
      matchLabels:
        environment: production
    kustomize:
      dir: overlays/production

  - name: dev
    clusterSelector:
      matchLabels:
        environment: development
    kustomize:
      dir: overlays/dev
```

## Step 7: Monitor Fleet Sync Status

```bash
# Check all GitRepos
kubectl get gitrepo -A

# Check Bundle deployment status
kubectl get bundle -n fleet-default
kubectl get bundledeployment -A

# Watch for errors
kubectl describe gitrepo -n fleet-default my-application | grep -A10 "Status:"

# Get detailed status per cluster
kubectl get bundledeployment -A -o wide
```

```bash
# From the Rancher UI:
# Continuous Delivery → Git Repos → select your repo → check "Clusters" tab
```

## Step 8: Set Up Webhook Triggers for Instant Sync

Instead of polling, trigger Fleet syncs immediately on git push:

```bash
# Get the Fleet webhook URL for the GitRepo
kubectl get gitrepo -n fleet-default my-application \
  -o jsonpath='{.status.webhookURL}'

# Configure this URL as a webhook in GitHub:
# Repository → Settings → Webhooks → Add Webhook
# Payload URL: <fleet-webhook-url>
# Content type: application/json
# Events: Push events
```

## Conclusion

Rancher Fleet provides a scalable, Git-native approach to deploying and managing workloads across Kubernetes clusters. By storing all Kubernetes manifests in Git and letting Fleet continuously reconcile state, you achieve auditability, repeatability, and drift prevention without additional CI/CD tooling. Fleet's cluster selector model makes it straightforward to target the right workloads to the right clusters at scale.
