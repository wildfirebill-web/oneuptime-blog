# How to Configure Fleet with Branch-Based Deployments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Fleet, GitOps, Branch Strategy, Kubernetes, Continuous Deployment

Description: Configure Rancher Fleet to deploy different versions of applications from different Git branches, enabling environment promotion workflows where staging and production track separate branches.

## Introduction

Branch-based deployment is a GitOps pattern where different Kubernetes environments track different Git branches. `main` deploys to production, `staging` to the staging cluster, and `develop` to development. Fleet's `GitRepo` resource makes branch-based deployments simple to configure and maintain, providing clear environment isolation through Git branching strategy.

## Branch Strategy

```
Git Repository:
├── main          → Production clusters
├── staging       → Staging cluster
└── develop       → Development cluster

Each branch contains:
├── frontend/
│   ├── fleet.yaml
│   └── deployment.yaml
├── backend/
│   ├── fleet.yaml
│   └── deployment.yaml
└── fleet.yaml    (root bundle config)
```

## Step 1: Create GitRepo Resources per Branch

```yaml
# gitrepo-production.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: myapp-production
  namespace: fleet-default
spec:
  repo: https://github.com/myorg/myapp-config.git
  branch: main                    # Track the main branch
  targets:
    - name: production
      clusterSelector:
        matchLabels:
          env: production
---
# gitrepo-staging.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: myapp-staging
  namespace: fleet-default
spec:
  repo: https://github.com/myorg/myapp-config.git
  branch: staging                 # Track the staging branch
  targets:
    - name: staging
      clusterSelector:
        matchLabels:
          env: staging
---
# gitrepo-develop.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: myapp-develop
  namespace: fleet-default
spec:
  repo: https://github.com/myorg/myapp-config.git
  branch: develop                 # Track the develop branch
  targets:
    - name: development
      clusterSelector:
        matchLabels:
          env: development
```

## Step 2: Label Your Clusters

```bash
# Label production cluster
kubectl label cluster production-cluster env=production \
  -n fleet-default

# Label staging cluster
kubectl label cluster staging-cluster env=staging \
  -n fleet-default

# Label development cluster
kubectl label cluster dev-cluster env=development \
  -n fleet-default
```

## Step 3: Configure Branch-Specific Values

Use Fleet's `values` override per GitRepo to customize deployments per branch:

```yaml
# gitrepo-production.yaml with overrides
spec:
  repo: https://github.com/myorg/myapp-config.git
  branch: main
  helm:
    values:
      replicaCount: 3
      resources:
        requests:
          memory: "512Mi"
          cpu: "500m"
      ingress:
        host: app.company.com
      image:
        pullPolicy: Always
  targets:
    - name: production
      clusterSelector:
        matchLabels:
          env: production
```

```yaml
# gitrepo-staging.yaml with overrides
spec:
  repo: https://github.com/myorg/myapp-config.git
  branch: staging
  helm:
    values:
      replicaCount: 1
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"
      ingress:
        host: staging.app.company.com
  targets:
    - name: staging
      clusterSelector:
        matchLabels:
          env: staging
```

## Step 4: Implement Promotion Workflow

Promote from develop to staging to production via Git merges:

```bash
# Promote develop → staging
git checkout staging
git merge develop
git push origin staging

# Fleet automatically deploys the merged changes to staging cluster
# After testing, promote staging → production
git checkout main
git merge staging
git push origin main

# Fleet automatically deploys to production cluster
```

## Step 5: Monitor Branch Deployments

```bash
# Check status of each branch deployment
kubectl get gitrepos -n fleet-default

# View detailed status
kubectl describe gitrepo myapp-production -n fleet-default

# Check bundle deployment status across branches
kubectl get bundles -n fleet-default

# View bundle deployments on each cluster
kubectl get bundledeployments -A
```

## Step 6: Configure Branch Protection

Protect the `main` branch with required status checks in GitHub/GitLab to ensure only tested code reaches production:

```yaml
# .github/branch-protection (conceptual)
# Configure via repository settings:
# - Require PR reviews before merging to main
# - Require status checks to pass
# - Restrict who can push directly to main
# - Require linear history
```

## Conclusion

Branch-based deployments with Rancher Fleet provide a clean, auditable path from development to production. Each environment tracks a specific branch, and promotions happen via Git merges rather than manual deployments. This pattern aligns with GitOps principles—the Git repository is the single source of truth, and all environment changes are tracked via pull requests and branch history.
