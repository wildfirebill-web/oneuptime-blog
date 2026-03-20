# How to Debug Fleet Git Repository Sync Issues - Part 2

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Fleet, GitOps, Debugging, Rancher, Kubernetes, Git Sync, SUSE Rancher

Description: Learn how to diagnose and fix Fleet Git repository sync failures including authentication errors, network issues, manifest parsing failures, and bundle deployment failures.

---

Fleet sync failures prevent GitOps updates from reaching your clusters. Issues range from Git authentication errors to Helm chart rendering failures. This guide walks through systematic diagnosis for each failure type.

---

## Fleet Sync Flow

```text
GitRepo (spec.repo)
    │
    ▼
Fleet Manager pulls Git repo
    │
    ▼
Fleet creates Bundle (rendered manifests)
    │
    ▼
Fleet Agent deploys Bundle to target clusters
    │
    ▼
BundleDeployment status updated
```

---

## Step 1: Check GitRepo Status

```bash
# List all GitRepos and their sync status

kubectl get gitrepo -n fleet-default

# Describe a specific GitRepo for error details
kubectl describe gitrepo my-app -n fleet-default

# Look for conditions:
# Ready: True/False
# Message: error details if sync failed
```

Common status conditions:
- `Synced` - Repository was cloned/fetched successfully
- `Ready` - All bundles are deployed
- `Error` - Sync failed (check Message field)

---

## Step 2: Check Fleet Manager Logs

```bash
# View Fleet manager logs for sync errors
kubectl logs -n cattle-fleet-system \
  deployment/fleet-controller \
  | grep -i "error\|failed\|sync" | tail -50

# Follow logs in real-time
kubectl logs -n cattle-fleet-system \
  deployment/fleet-controller -f
```

---

## Common Issue 1: Git Authentication Failure

```bash
# Check if the GitRepo references a secret for auth
kubectl get gitrepo my-app -n fleet-default \
  -o jsonpath='{.spec.clientSecretName}'

# Verify the secret exists and has the correct keys
kubectl get secret my-git-auth -n fleet-default -o yaml

# For SSH authentication, keys should be:
# - ssh-privatekey
# - known_hosts

# For HTTP/HTTPS, keys should be:
# - username
# - password

# Create the secret if missing
kubectl create secret generic my-git-auth \
  -n fleet-default \
  --from-literal=username=git \
  --from-literal=password=my-token

# Update the GitRepo to use the secret
kubectl patch gitrepo my-app -n fleet-default \
  --type merge \
  -p '{"spec":{"clientSecretName":"my-git-auth"}}'
```

---

## Common Issue 2: Network Connectivity to Git Repository

```bash
# Test connectivity from the Fleet manager pod
kubectl exec -n cattle-fleet-system \
  $(kubectl get pod -n cattle-fleet-system \
    -l app=fleet-controller -o name | head -1) \
  -- curl -v https://github.com

# For private Git servers, test the specific URL
kubectl exec -n cattle-fleet-system \
  $(kubectl get pod -n cattle-fleet-system \
    -l app=fleet-controller -o name | head -1) \
  -- curl -v https://git.example.com
```

---

## Common Issue 3: Helm Chart Rendering Failures

```bash
# Check the Bundle for rendering errors
kubectl get bundle -n fleet-default | grep -v "1/1"

# Describe the failing bundle
kubectl describe bundle my-app -n fleet-default

# Look for the status message - it shows the Helm template error
# Example: "Error: template: my-chart/templates/deployment.yaml:15:5: executing ..."
```

---

## Common Issue 4: Target Cluster Not Matching

```bash
# Check if any clusters match the GitRepo's target selector
kubectl get gitrepo my-app -n fleet-default \
  -o jsonpath='{.spec.targets}'

# List available clusters and their labels
kubectl get cluster -n fleet-default \
  -o custom-columns='NAME:.metadata.name,LABELS:.metadata.labels'

# If no clusters match the selector, the GitRepo shows 0/0 in cluster counts
# Fix: add the required label to the target cluster
kubectl label cluster my-cluster -n fleet-default env=production
```

---

## Common Issue 5: BundleDeployment Failures

```bash
# Check BundleDeployments for failure reasons
kubectl get bundledeployment -n fleet-default

# Describe a failing deployment
kubectl describe bundledeployment my-app-abc123 -n fleet-default

# Check Fleet agent logs on the target cluster
# (Run this on the downstream cluster, not the management cluster)
kubectl logs -n cattle-fleet-system \
  deployment/fleet-agent -f \
  | grep -i "error\|failed"
```

---

## Step 3: Force a Resync

```bash
# Increment the forceSyncGeneration to force a re-pull from Git
kubectl patch gitrepo my-app -n fleet-default \
  --type merge \
  -p '{"spec":{"forceSyncGeneration":1}}'

# Or increment again if already set
CURRENT=$(kubectl get gitrepo my-app -n fleet-default \
  -o jsonpath='{.spec.forceSyncGeneration}')
kubectl patch gitrepo my-app -n fleet-default \
  --type merge \
  -p "{\"spec\":{\"forceSyncGeneration\":$((CURRENT+1))}}"
```

---

## Step 4: Check the Fleet Agent on Downstream Clusters

```bash
# Verify Fleet agent is running on the downstream cluster
kubectl get pods -n cattle-fleet-system

# Check for connection errors
kubectl logs -n cattle-fleet-system deployment/fleet-agent \
  | grep -i "error\|connect\|refused"

# Restart the Fleet agent if it's stuck
kubectl rollout restart deployment/fleet-agent -n cattle-fleet-system
```

---

## Debugging Checklist

```bash
# 1. Is the GitRepo able to clone the repository?
kubectl describe gitrepo <name> -n fleet-default | grep -A 5 Conditions

# 2. Are Bundles being created?
kubectl get bundle -n fleet-default | grep <gitrepo-name>

# 3. Are BundleDeployments being applied?
kubectl get bundledeployment -n fleet-default

# 4. Is the Fleet agent running on downstream clusters?
# (on each downstream cluster)
kubectl get pods -n cattle-fleet-system

# 5. Are there RBAC issues on downstream clusters?
kubectl get clusterrolebinding | grep fleet
```

---

## Best Practices

- Use HTTPS with token authentication for Git repos rather than SSH - it's easier to debug and doesn't require managing SSH keys.
- Enable Fleet webhook integration with your Git provider to trigger immediate syncs on push rather than waiting for the polling interval.
- Test `fleet.yaml` and Helm templates locally using `fleet apply --dry-run` before pushing to Git - this catches rendering errors without requiring a cluster sync.
