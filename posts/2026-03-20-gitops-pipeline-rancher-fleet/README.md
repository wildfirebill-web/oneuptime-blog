# How to Build a GitOps Pipeline with Rancher Fleet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Fleet, GitOps, Kubernetes, CI/CD, GitHub Action, Automation

Description: Learn how to build an end-to-end GitOps pipeline using Rancher Fleet to automatically deploy Kubernetes workloads from a Git repository to multiple clusters.

---

Rancher Fleet is a GitOps engine built into Rancher that continuously synchronizes Kubernetes cluster state from Git. This guide walks through building a complete GitOps pipeline: from writing manifests in Git to automatic deployment across environments.

---

## Architecture

```text
Developer pushes to Git
         ↓
GitHub Actions runs tests
         ↓
Fleet agent detects change
         ↓
Fleet applies manifests to clusters
         ↓
Rancher shows deployment status
```

---

## Prerequisites

- Rancher 2.6+ with Fleet enabled
- One or more Kubernetes clusters registered in Rancher
- GitHub repository for manifests
- kubectl access to Rancher cluster

---

## Step 1: Label Your Clusters

Fleet uses cluster labels to target deployments:

```bash
# In Rancher UI: Cluster Management → Edit Cluster → Labels and Annotations

# Or via kubectl

kubectl label cluster.fleet.cattle.io/production \
    env=production \
    region=us-east-1 \
    -n fleet-default

kubectl label cluster.fleet.cattle.io/staging \
    env=staging \
    region=us-east-1 \
    -n fleet-default
```

---

## Step 2: Create Your Application Manifests

```yaml
# apps/myapp/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myorg/myapp:v1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
```

---

## Step 3: Create fleet.yaml

```yaml
# apps/myapp/fleet.yaml
defaultNamespace: production

targetCustomizations:
  - name: production
    clusterSelector:
      matchLabels:
        env: production
    yaml:
      overlays:
        - production

  - name: staging
    clusterSelector:
      matchLabels:
        env: staging
    yaml:
      overlays:
        - staging

diff:
  comparePatches:
    - apiVersion: apps/v1
      kind: Deployment
      name: myapp
      namespace: production
      operations:
        - {"op": "remove", "path": "/spec/template/metadata/annotations"}
```

---

## Step 4: Register the GitRepo in Rancher Fleet

```yaml
# gitrepo.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: myapp
  namespace: fleet-default
spec:
  repo: https://github.com/myorg/gitops-repo
  branch: main
  paths:
    - apps/myapp
  targets:
    - name: production
      clusterSelector:
        matchLabels:
          env: production
    - name: staging
      clusterSelector:
        matchLabels:
          env: staging
```

```bash
kubectl apply -f gitrepo.yaml
```

---

## Step 5: Configure Git Credentials

```bash
# Create secret for private repository
kubectl create secret generic github-credentials \
    --from-literal=username=myorg \
    --from-literal=password=$GITHUB_TOKEN \
    -n fleet-default

# Update gitrepo to use credentials
kubectl patch gitrepo myapp -n fleet-default \
    --type=merge \
    -p '{"spec":{"clientSecretName":"github-credentials"}}'
```

---

## Step 6: Build the CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy to Kubernetes

on:
  push:
    branches:
      - main
      - staging

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Validate manifests
        run: |
          kubectl kustomize apps/myapp > /dev/null
          echo "Manifests are valid"

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: myorg/myapp:${{ github.sha }}

      - name: Update image tag in manifests
        run: |
          cd apps/myapp/overlays/production
          kustomize edit set image myorg/myapp:${{ github.sha }}
          git config user.email "ci@myorg.com"
          git config user.name "CI Bot"
          git add -A
          git commit -m "deploy: update myapp to ${{ github.sha }}"
          git push
```

---

## Step 7: Monitor Deployments

```bash
# Check GitRepo sync status
kubectl get gitrepo -n fleet-default

# Check bundle status (bundles = fleet deployment units)
kubectl get bundle -n fleet-default

# Check individual bundle deployments
kubectl get bundledeployment -A

# Detailed status
kubectl describe gitrepo myapp -n fleet-default
```

### Rancher UI Monitoring

1. Open Rancher → **Continuous Delivery** (Fleet)
2. Select **Git Repos** - see sync status for each repo
3. Select **Bundles** - see per-cluster deployment status
4. Red = out of sync or error, Green = deployed successfully

---

## Step 8: Rollback

```bash
# Find previous commit
git log --oneline apps/myapp/

# Revert to previous version
git revert HEAD
git push origin main

# Fleet automatically detects the new commit and rolls back
```

---

## Best Practices

1. **Use tags not branches** for production deployments - tags are immutable
2. **Validate manifests in CI** before they reach Fleet
3. **Monitor bundle status** with alerting - failed bundles should page on-call
4. **Use `prune: true`** to automatically clean up deleted resources
5. **Separate config repos** from app code repos for cleaner history

---

## Conclusion

Rancher Fleet provides a complete GitOps pipeline with minimal setup. Register your Git repository as a GitRepo, label your clusters, and Fleet handles continuous synchronization. Combine with GitHub Actions for image building and tag updates to create a fully automated, end-to-end delivery pipeline.

---

*Monitor your Fleet deployments and cluster health with [OneUptime](https://oneuptime.com).*
