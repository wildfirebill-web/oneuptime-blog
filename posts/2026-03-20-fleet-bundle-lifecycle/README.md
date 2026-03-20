# How to Manage Fleet Bundle Lifecycle

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Fleet, GitOps, Rancher, Kubernetes, Bundles

Description: Learn how to manage the complete lifecycle of Fleet bundles, from creation through updates, pausing, and deletion, with best practices for production environments.

## Introduction

Fleet Bundles are the internal deployment units that Fleet creates from your Git repositories. Each directory in a GitRepo that contains Kubernetes manifests or a fleet.yaml becomes a Bundle. Understanding how bundles are created, updated, and deleted is essential for managing your Fleet deployments reliably.

This guide covers the complete bundle lifecycle: creation, status monitoring, updating, pausing, and cleanup.

## Prerequisites

- Fleet installed and configured
- Existing GitRepo resources
- `kubectl` access to Fleet manager

## Understanding the Bundle Lifecycle

The lifecycle of a Fleet Bundle follows these stages:

1. **Creation**: Fleet discovers a new path in a GitRepo and creates a Bundle
2. **Processing**: Fleet evaluates targets and creates BundleDeployments
3. **Deployment**: Fleet agents on downstream clusters apply the resources
4. **Monitoring**: Fleet continuously checks the deployed state
5. **Update**: A new Git commit triggers a Bundle update
6. **Deletion**: GitRepo deletion triggers Bundle and resource cleanup

## Viewing Bundle Status

```bash
# List all bundles across all namespaces
kubectl get bundles -A

# List bundles in a specific workspace
kubectl get bundles -n fleet-default

# Get detailed bundle information
kubectl get bundle my-app -n fleet-default -o yaml

# View bundle status summary
kubectl get bundles -n fleet-default \
  -o custom-columns=\
'NAME:.metadata.name,\
READY:.status.summary.ready,\
DESIRED:.status.summary.desiredReady,\
NOT_READY:.status.summary.notReady,\
MODIFIED:.status.summary.modified'
```

### Bundle Status Fields

```bash
# Get the detailed status of a bundle
kubectl get bundle my-app -n fleet-default \
  -o jsonpath='{.status}' | python3 -m json.tool
```

Key status fields:
- `summary.ready`: Number of clusters with bundle deployed successfully
- `summary.desiredReady`: Total clusters that should have the bundle
- `summary.notReady`: Clusters where deployment failed
- `summary.modified`: Clusters where deployed state differs from desired
- `summary.waitApplied`: Clusters waiting for deployment

## Managing BundleDeployments

Each cluster that receives a bundle gets a `BundleDeployment` resource:

```bash
# List all bundle deployments across all namespaces
kubectl get bundledeployments -A

# Filter bundle deployments for a specific bundle
kubectl get bundledeployments -A \
  -o jsonpath='{range .items[?(@.spec.bundleID=="fleet-default/my-app")]}{.metadata.name}{"\n"}{end}'

# Check a specific bundle deployment
kubectl describe bundledeployment my-app -n my-cluster-namespace
```

## Pausing a Bundle

To temporarily stop Fleet from updating a bundle (for maintenance or testing):

```bash
# Pause a bundle by patching the GitRepo
# This sets a specific commit, freezing the deployment
CURRENT_COMMIT=$(kubectl get gitrepo my-app -n fleet-default \
  -o jsonpath='{.status.commit}')

echo "Current commit: ${CURRENT_COMMIT}"

# Pin to the current commit to freeze updates
kubectl patch gitrepo my-app -n fleet-default \
  --type=merge \
  -p "{\"spec\":{\"revision\":\"${CURRENT_COMMIT}\"}}"

echo "Bundle updates are now paused at commit: ${CURRENT_COMMIT}"
```

### Resuming a Paused Bundle

```bash
# Resume auto-updates by switching back to branch tracking
kubectl patch gitrepo my-app -n fleet-default \
  --type=merge \
  -p '{"spec":{"revision":"","branch":"main"}}'

echo "Bundle updates resumed - tracking main branch"
```

## Forcing a Bundle Re-Deployment

To force Fleet to re-apply all resources even if no Git changes occurred:

```bash
# Delete and recreate the bundle (causes brief downtime)
# Better approach: annotate the GitRepo
kubectl annotate gitrepo my-app \
  -n fleet-default \
  fleet.cattle.io/commit="" \
  --overwrite

# Wait for Fleet to re-sync
kubectl get gitrepo my-app -n fleet-default -w
```

## Deleting Bundles

### Automatic Deletion via GitRepo Removal

When you delete a GitRepo, Fleet automatically removes all associated Bundles and deployed resources:

```bash
# Delete the GitRepo - this triggers bundle and resource cleanup
kubectl delete gitrepo my-app -n fleet-default

# Verify bundles are being removed
kubectl get bundles -n fleet-default -w
```

### Keeping Resources After Bundle Deletion

To delete bundles without removing deployed resources:

```yaml
# gitrepo-keep-resources.yaml
spec:
  # When this GitRepo is deleted, keep the deployed resources
  keepResources: true
```

```bash
# Apply the keepResources setting before deleting
kubectl patch gitrepo my-app -n fleet-default \
  --type=merge \
  -p '{"spec":{"keepResources":true}}'

# Now delete the GitRepo
kubectl delete gitrepo my-app -n fleet-default
# Resources remain on clusters
```

## Bundle Update History

Fleet doesn't maintain a built-in rollback history, but you can track it via Git:

```bash
# View the current commit Fleet is using
kubectl get gitrepo my-app -n fleet-default \
  -o jsonpath='{.status.commit}'

# View the Git log to find previous commits
git -C /path/to/repo log --oneline -20

# Roll back by pinning to a previous commit
kubectl patch gitrepo my-app -n fleet-default \
  --type=merge \
  -p '{"spec":{"revision":"abc1234def5678"}}'
```

## Monitoring Bundle Health in Production

```bash
# One-liner to check overall fleet health
kubectl get bundles -A \
  -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}: ready={.status.summary.ready}, desired={.status.summary.desiredReady}{"\n"}{end}'

# Find all non-ready bundles
kubectl get bundles -A \
  --field-selector 'status.summary.ready!=status.summary.desiredReady' \
  2>/dev/null || \
  kubectl get bundles -A \
  -o jsonpath='{range .items[?(@.status.summary.ready!=@.status.summary.desiredReady)]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}'

# Check events for bundle errors
kubectl get events -A \
  --field-selector reason=FailedSync \
  --sort-by='.lastTimestamp'
```

## Conclusion

Managing the Fleet bundle lifecycle effectively is key to maintaining a reliable GitOps deployment system. By understanding how bundles are created from GitRepo paths, how they propagate to clusters as BundleDeployments, and how to safely pause, update, and remove them, you can maintain full control over your deployment pipeline. Combine lifecycle management with proper monitoring to ensure your Fleet deployments remain healthy and synchronized with your Git repository at all times.
