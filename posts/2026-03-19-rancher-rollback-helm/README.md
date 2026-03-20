# How to Roll Back Helm Releases in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Helm

Description: Learn how to roll back Helm releases in Rancher to recover from failed upgrades, configuration errors, or unstable deployments.

When a Helm upgrade goes wrong, whether due to a misconfigured value, an incompatible chart version, or a buggy application release, you need to quickly roll back to a known good state. Rancher and Helm both provide mechanisms to roll back releases to previous revisions. This guide covers all the methods and best practices for rolling back Helm releases.

## Prerequisites

- A running Rancher instance (v2.7 or later)
- A managed Kubernetes cluster with Helm releases that have been upgraded at least once
- Access to the namespace containing the releases

## Understanding Helm Rollbacks

When you roll back a Helm release, Helm:

1. Retrieves the manifest and values from the target revision
2. Renders the chart templates with those values
3. Applies the resulting manifests to the cluster
4. Creates a new revision (the rollback itself is a new revision)

For example, if you roll back from revision 4 to revision 2, you get revision 5 that matches the state of revision 2.

## Step 1: Identify the Problem

Before rolling back, confirm the upgrade caused the issue:

```bash
# Check release status

helm status my-app -n default

# View release history
helm history my-app -n default

# Check pod status
kubectl get pods -l app.kubernetes.io/instance=my-app -n default

# View pod events
kubectl describe pods -l app.kubernetes.io/instance=my-app -n default

# Check logs
kubectl logs -l app.kubernetes.io/instance=my-app -n default --tail=50
```

In the Rancher UI, navigate to **Apps > Installed Apps** and check the release status. Then go to **Workloads** to inspect pod health.

## Step 2: Roll Back via the Rancher UI

1. Navigate to **Apps > Installed Apps**
2. Find the release you want to roll back
3. Click the three-dot menu and select **Rollback**
4. Select the target revision from the dropdown list
5. Click **Rollback** to confirm

Rancher will execute the rollback and show you the progress. The release status will update once the rollback completes.

## Step 3: Roll Back via the Helm CLI

### Roll Back to the Previous Revision

```bash
helm rollback my-app -n default
```

Without specifying a revision, Helm rolls back to the immediately previous revision.

### Roll Back to a Specific Revision

First, view the history to find the target revision:

```bash
helm history my-app -n default
```

Then roll back to the desired revision:

```bash
helm rollback my-app 2 -n default
```

### Roll Back with Options

```bash
# Wait for rollback to complete
helm rollback my-app 2 -n default --wait --timeout 5m

# Force resource updates even if conflicts exist
helm rollback my-app 2 -n default --force

# Clean up new resources created during the failed upgrade
helm rollback my-app 2 -n default --cleanup-on-fail
```

## Step 4: Verify the Rollback

After the rollback completes:

```bash
# Check the release status
helm status my-app -n default

# Verify the revision
helm history my-app -n default

# Check the values match the expected revision
helm get values my-app -n default

# Verify pods are healthy
kubectl get pods -l app.kubernetes.io/instance=my-app -n default

# Test the application
kubectl port-forward svc/my-app 8080:80 -n default
```

In Rancher, navigate to **Apps > Installed Apps** and verify the release shows as "deployed". Check **Workloads** to confirm all pods are running.

## Handling Special Rollback Scenarios

### Rolling Back with Persistent Data

If the chart manages databases or other stateful services, be aware that:

- Schema migrations applied during the upgrade are not automatically reverted
- Data written by the new version may not be compatible with the old version
- PersistentVolumeClaims are not affected by Helm rollbacks

Before rolling back stateful applications:

1. Take a backup of your data
2. Check if the application supports downgrade migrations
3. Test the rollback in a staging environment first

### Rolling Back When Pods Are Stuck

If new pods are stuck in a crash loop and old pods have been terminated:

```bash
# Force the rollback
helm rollback my-app 2 -n default --force

# If pods are still stuck, check for PVC issues
kubectl get pvc -l app.kubernetes.io/instance=my-app -n default

# Check for resource quota issues
kubectl describe namespace default | grep -A 10 "Resource Quotas"
```

### Rolling Back Custom Resource Definitions

Helm does not delete CRDs during rollback. If a chart upgrade added new CRDs, they remain after rollback. This is usually harmless, but if the CRDs cause issues:

```bash
# List CRDs related to the chart
kubectl get crds | grep my-app

# Manually remove if needed (be careful - this deletes all instances)
kubectl delete crd myresource.example.com
```

### Rolling Back with Hook Failures

If Helm hooks (pre-install, post-install, etc.) fail during rollback:

```bash
# Skip hooks during rollback
helm rollback my-app 2 -n default --no-hooks
```

Only use this if you understand the impact of skipping hooks.

## Automating Rollbacks

### Using --atomic on Upgrades

The best way to handle rollbacks is to prevent the need for manual intervention:

```bash
helm upgrade my-app my-chart \
  --version 2.0.0 \
  --namespace default \
  -f values.yaml \
  --atomic \
  --timeout 5m
```

The `--atomic` flag automatically rolls back if the upgrade fails or times out.

### CI/CD Pipeline Rollback

In a CI/CD pipeline, automate rollback on failure:

```bash
#!/bin/bash
set -e

# Attempt the upgrade
if ! helm upgrade my-app my-chart \
  --version $CHART_VERSION \
  --namespace default \
  -f values.yaml \
  --wait \
  --timeout 5m; then

  echo "Upgrade failed, rolling back..."
  helm rollback my-app -n default --wait --timeout 5m
  exit 1
fi

echo "Upgrade successful"
```

### Health Check After Upgrade

```bash
#!/bin/bash

# Upgrade
helm upgrade my-app my-chart --version 2.0.0 -n default -f values.yaml --wait

# Wait for application health
sleep 30

# Check application health
if ! curl -sf http://my-app-service/health; then
  echo "Health check failed, rolling back..."
  helm rollback my-app -n default --wait
  exit 1
fi
```

## Rolling Back in Multi-Chart Environments

When multiple charts depend on each other, plan the rollback order:

1. Roll back dependent charts first (e.g., application before database)
2. Verify each rollback before proceeding
3. Consider the impact on inter-service communication

```bash
# Roll back in reverse dependency order
helm rollback frontend 3 -n default --wait
helm rollback backend 5 -n default --wait
helm rollback database 2 -n default --wait
```

## Best Practices

1. Always keep enough revision history for meaningful rollbacks (`--history-max 10` or more)
2. Use `--atomic` for production upgrades to enable automatic rollback
3. Test rollback procedures regularly in staging environments
4. Document the rollback process for your team
5. Back up data before rolling back stateful applications
6. Use `helm diff` to compare the current state with the target revision before rolling back
7. Monitor application health immediately after a rollback

## Summary

Rolling back Helm releases in Rancher is straightforward using either the UI or the Helm CLI. The key is to maintain sufficient revision history, verify the target revision before rolling back, and confirm application health after the rollback completes. For production environments, use the `--atomic` flag on upgrades to enable automatic rollbacks, and always test rollback procedures in staging before relying on them in production.
