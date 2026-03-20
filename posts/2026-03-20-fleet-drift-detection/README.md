# How to Configure Fleet Drift Detection

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Fleet, GitOps, Rancher, Kubernetes, Drift Detection

Description: Learn how to configure Fleet's drift detection to automatically identify and remediate configuration drift in your Kubernetes clusters.

## Introduction

Configuration drift occurs when the actual state of your Kubernetes resources diverges from the desired state defined in Git. Drift can happen when someone manually edits resources, when an operator modifies a deployment, or when a cluster partially applies updates. Fleet's drift detection feature continuously monitors your clusters and automatically reconciles any detected drift.

This guide covers how to enable and configure Fleet's drift detection, understand drift conditions, and respond to drift events.

## Prerequisites

- Fleet installed in Rancher (v0.6+ for drift detection support)
- GitRepo resources configured
- `kubectl` access to Fleet manager

## How Fleet Drift Detection Works

Fleet's agent in each downstream cluster periodically compares the live state of deployed resources against the desired state stored in the bundle. When it detects a difference (drift), it can:

1. **Report** the drift (non-remediation mode)
2. **Remediate** the drift by re-applying the desired state

## Enabling Drift Detection

Drift detection is configured at the GitRepo level:

```yaml
# gitrepo-drift-detection.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: my-app
  namespace: fleet-default
spec:
  repo: https://github.com/my-org/my-app
  branch: main

  # Enable drift detection and correction
  correctDrift:
    # Enable drift correction (re-applies desired state when drift detected)
    enabled: true

    # Force correction even for resources not owned by Fleet
    # Use with caution in shared clusters
    force: false

    # Keep resources that exist in the cluster but not in Git
    # (true = don't delete extra resources)
    keepResources: false

  targets:
    - clusterSelector: {}
```

```bash
# Apply the GitRepo with drift detection
kubectl apply -f gitrepo-drift-detection.yaml

# Verify drift detection is configured
kubectl get gitrepo my-app -n fleet-default -o jsonpath='{.spec.correctDrift}'
```

## Testing Drift Detection

To test that drift detection is working:

```bash
# Step 1: Deploy an application via Fleet and confirm it's running
kubectl get bundle my-app -n fleet-default

# Step 2: Manually scale the deployment on a downstream cluster
# (This simulates drift)
kubectl scale deployment my-app --replicas=5 -n my-app

# Step 3: Wait for Fleet's drift detection interval (typically 15 seconds)
# Step 4: Verify Fleet corrected the drift
kubectl get deployment my-app -n my-app -o jsonpath='{.spec.replicas}'
# Should return to the value defined in Git
```

## Monitoring Drift Status

### Checking Drift in Bundle Deployments

```bash
# List all bundle deployments and check drift status
kubectl get bundledeployments -A

# Check a specific bundle deployment for drift
kubectl describe bundledeployment my-app -n fleet-default

# Look for "Modified" status indicating drift was detected
kubectl get bundledeployments -A \
  -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.modifiedStatus}{"\n"}{end}'
```

### GitRepo Drift Status Conditions

```bash
# Check the GitRepo conditions for drift information
kubectl get gitrepo my-app -n fleet-default \
  -o jsonpath='{.status.conditions}' | python3 -m json.tool

# Watch for drift-related events
kubectl get events -n fleet-default \
  --field-selector reason=Modified \
  --sort-by='.lastTimestamp'
```

## Configuring Drift Detection Per Target

You can configure drift detection differently for different cluster types:

```yaml
# fleet.yaml - Per-target drift behavior
targets:
  # Production: always correct drift immediately
  - name: production
    clusterSelector:
      matchLabels:
        env: production
    correctDrift:
      enabled: true
      force: false

  # Development: report drift but don't auto-correct
  # Developers may want to experiment with live changes
  - name: development
    clusterSelector:
      matchLabels:
        env: dev
    correctDrift:
      enabled: false
```

## Understanding Modified Status

When Fleet detects drift, the bundle deployment shows `Modified` status. The `modifiedStatus` field provides details:

```bash
# Get detailed modified status
kubectl get bundledeployment my-app-local -n fleet-local \
  -o jsonpath='{.status.modifiedStatus}' | python3 -m json.tool
```

This shows output like:
```json
[
  {
    "apiVersion": "apps/v1",
    "kind": "Deployment",
    "name": "my-app",
    "namespace": "my-app",
    "patch": "{\"spec\":{\"replicas\":5}}"
  }
]
```

## Handling Drift in Shared Clusters

In clusters where multiple teams deploy resources, be careful with drift correction:

```yaml
# gitrepo-shared-cluster.yaml
spec:
  correctDrift:
    enabled: true
    # Do NOT force correction - avoid overwriting other teams' changes
    force: false
    # Keep resources that exist but aren't in our Git repo
    keepResources: true
```

## Drift Detection for Immutable Fields

Some Kubernetes fields are immutable after creation (like Job selectors). Fleet handles this gracefully:

```bash
# Check for immutable field errors
kubectl get events -n fleet-default \
  --field-selector reason=FailedSync
```

## Setting Up Alerts for Drift

Integrate drift detection with your monitoring stack by watching for condition changes:

```bash
# Script to report all clusters with drift
kubectl get bundledeployments -A \
  -o jsonpath='{range .items[?(@.status.modified==true)]}{.metadata.namespace}{" "}{.metadata.name}{"\n"}{end}'
```

## Conclusion

Fleet's drift detection provides continuous compliance enforcement for your Kubernetes clusters. By automatically detecting and correcting deviations from the Git-defined desired state, you maintain consistency across your entire fleet without manual intervention. Configure drift detection aggressively for production environments where changes outside of GitOps workflows should not persist, and more permissively for development environments where experimentation is expected. Combined with proper monitoring and alerting, drift detection gives you confidence that your clusters always reflect exactly what is defined in Git.
