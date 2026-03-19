# How to Troubleshoot Failed Helm Deployments in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Helm

Description: Learn how to diagnose and fix common Helm deployment failures in Rancher, from image pull errors to resource conflicts and hook failures.

Helm deployments can fail for many reasons, including incorrect values, missing resources, image pull errors, insufficient cluster resources, and hook failures. Diagnosing these failures quickly is essential for maintaining service reliability. This guide covers the most common Helm deployment failures in Rancher and how to resolve them.

## Prerequisites

- A running Rancher instance (v2.7 or later)
- A managed Kubernetes cluster with Helm releases
- Access to the namespace where the failed release was deployed
- Familiarity with kubectl and Helm CLI

## Step 1: Identify the Failure

### Check Release Status in Rancher

1. Navigate to **Apps > Installed Apps**
2. Look for releases with a **Failed** status
3. Click on the failed release to see error details

### Check Release Status via CLI

```bash
helm list -n default --all
```

The `--all` flag shows releases in all states, including failed ones.

```bash
helm status my-app -n default
```

This shows the release status and any error messages.

### View Release History

```bash
helm history my-app -n default
```

Look for the revision that failed and its description.

## Step 2: Common Failure Scenarios

### Scenario 1: Image Pull Errors

**Symptoms**: Pods stuck in `ImagePullBackOff` or `ErrImagePull` state.

**Diagnose**:

```bash
kubectl get pods -l app.kubernetes.io/instance=my-app -n default
kubectl describe pod <pod-name> -n default
```

Look for events like:

```
Failed to pull image "my-app:v2.0": rpc error: code = Unknown desc = Error response from daemon: manifest for my-app:v2.0 not found
```

**Fix**:

1. Verify the image exists in the registry:
   ```bash
   docker manifest inspect my-registry.example.com/my-app:v2.0
   ```

2. Check image pull secrets:
   ```bash
   kubectl get secrets -n default | grep pull
   ```

3. Create or update the pull secret if needed:
   ```bash
   kubectl create secret docker-registry my-pull-secret \
     --docker-server=my-registry.example.com \
     --docker-username=user \
     --docker-password=pass \
     -n default
   ```

4. Add the pull secret to your Helm values:
   ```yaml
   imagePullSecrets:
     - name: my-pull-secret
   ```

### Scenario 2: Insufficient Resources

**Symptoms**: Pods stuck in `Pending` state.

**Diagnose**:

```bash
kubectl describe pod <pod-name> -n default
```

Look for events like:

```
0/3 nodes are available: 3 Insufficient memory. 3 Insufficient cpu.
```

**Fix**:

1. Check cluster capacity:
   ```bash
   kubectl top nodes
   kubectl describe nodes | grep -A 5 "Allocated resources"
   ```

2. Reduce resource requests in chart values:
   ```yaml
   resources:
     requests:
       cpu: 100m
       memory: 128Mi
     limits:
       cpu: 500m
       memory: 256Mi
   ```

3. Or add more nodes to the cluster through Rancher's cluster management.

### Scenario 3: Resource Quota Exceeded

**Symptoms**: Pods fail to create with quota errors.

**Diagnose**:

```bash
kubectl describe namespace default | grep -A 20 "Resource Quotas"
kubectl get events -n default --sort-by=.lastTimestamp
```

**Fix**:

1. Check current quota usage:
   ```bash
   kubectl describe resourcequota -n default
   ```

2. Either increase the quota or reduce resource requests in the chart values.

### Scenario 4: PersistentVolumeClaim Issues

**Symptoms**: Pods stuck in `Pending` waiting for a PVC to bind.

**Diagnose**:

```bash
kubectl get pvc -n default
kubectl describe pvc <pvc-name> -n default
```

Look for:

```
waiting for first consumer to be created before binding
no persistent volumes available for this claim
```

**Fix**:

1. Check available StorageClasses:
   ```bash
   kubectl get storageclass
   ```

2. Set the correct StorageClass in the chart values:
   ```yaml
   persistence:
     storageClass: "standard"
   ```

3. Ensure the provisioner can create volumes in the requested size.

### Scenario 5: Helm Hook Failures

**Symptoms**: Release stuck in `pending-install` or `pending-upgrade` state.

**Diagnose**:

```bash
# Check for hook pods
kubectl get pods -n default | grep -E "(pre-install|post-install|pre-upgrade|post-upgrade)"

# View hook pod logs
kubectl logs <hook-pod-name> -n default

# Check hook job status
kubectl get jobs -n default
```

**Fix**:

1. Fix the hook script or image
2. Delete the failed hook resources:
   ```bash
   kubectl delete job <hook-job-name> -n default
   ```

3. If the release is stuck, force remove it:
   ```bash
   # Careful: this deletes the release metadata
   helm uninstall my-app -n default --no-hooks

   # Then reinstall
   helm install my-app my-chart -n default -f values.yaml
   ```

### Scenario 6: Template Rendering Errors

**Symptoms**: Installation fails immediately with a template error.

**Diagnose**:

```bash
# Try to render templates locally
helm template my-app my-chart -f values.yaml

# Or use dry run
helm install my-app my-chart --dry-run -f values.yaml -n default
```

Common errors:

```
Error: template: my-chart/templates/deployment.yaml:15:20: executing "my-chart/templates/deployment.yaml" at <.Values.image.tag>: nil pointer evaluating interface {}.tag
```

**Fix**:

1. Check that all required values are set
2. Verify the values YAML syntax is correct
3. Ensure you are using the correct chart version for your values

### Scenario 7: RBAC and Permission Errors

**Symptoms**: Pods fail with permission-denied errors or service accounts are missing.

**Diagnose**:

```bash
kubectl get serviceaccount -n default
kubectl get clusterrolebinding | grep my-app
kubectl describe pod <pod-name> -n default
```

**Fix**:

1. Ensure the chart creates the necessary ServiceAccount:
   ```yaml
   serviceAccount:
     create: true
     name: my-app
   ```

2. If using a pre-existing ServiceAccount, verify it exists and has the correct permissions.

### Scenario 8: CRD Conflicts

**Symptoms**: Installation fails with "resource already exists" errors.

**Diagnose**:

```bash
kubectl get crd | grep my-app
```

**Fix**:

1. If upgrading, CRDs from a previous installation may conflict:
   ```bash
   kubectl get crd <crd-name> -o yaml | grep -A 2 "labels"
   ```

2. Some charts require manual CRD updates:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/chart-repo/crds/main/crd.yaml
   ```

## Step 3: General Debugging Commands

A comprehensive set of commands for diagnosing any Helm failure:

```bash
# Release overview
helm status my-app -n default
helm history my-app -n default
helm get values my-app -n default
helm get manifest my-app -n default

# Pod status
kubectl get pods -l app.kubernetes.io/instance=my-app -n default -o wide
kubectl describe pods -l app.kubernetes.io/instance=my-app -n default

# Events (sorted by time)
kubectl get events -n default --sort-by=.lastTimestamp | tail -30

# Logs from all pods
kubectl logs -l app.kubernetes.io/instance=my-app -n default --all-containers --tail=50

# Services and endpoints
kubectl get svc -l app.kubernetes.io/instance=my-app -n default
kubectl get endpoints -l app.kubernetes.io/instance=my-app -n default

# ConfigMaps and Secrets
kubectl get configmap -l app.kubernetes.io/instance=my-app -n default
kubectl get secret -l app.kubernetes.io/instance=my-app -n default
```

## Step 4: Recovering from Failed States

### Delete and Reinstall

If the release is in a bad state and cannot be upgraded or rolled back:

```bash
# Uninstall the release
helm uninstall my-app -n default

# Clean up any remaining resources
kubectl delete all -l app.kubernetes.io/instance=my-app -n default
kubectl delete pvc -l app.kubernetes.io/instance=my-app -n default
kubectl delete configmap -l app.kubernetes.io/instance=my-app -n default
kubectl delete secret -l app.kubernetes.io/instance=my-app -n default

# Reinstall
helm install my-app my-chart -n default -f values.yaml
```

### Fix a Stuck Release

If a release is stuck in `pending-install` or `pending-upgrade`:

```bash
# Find the release secret
kubectl get secrets -n default -l owner=helm,name=my-app

# Delete the pending secret (the one with status=pending-*)
kubectl delete secret sh.helm.release.v1.my-app.v3 -n default

# Now you can retry the operation
helm upgrade my-app my-chart -n default -f values.yaml
```

## Best Practices for Preventing Failures

1. Always use `--dry-run` before installing or upgrading in production
2. Use `helm diff` to preview changes before applying them
3. Set `--atomic` for automatic rollback on failure
4. Configure proper health checks in your chart templates
5. Test chart deployments in a staging environment first
6. Keep chart values in version control
7. Set `--timeout` to a reasonable value to catch hanging deployments
8. Monitor Rancher's **Apps > Installed Apps** page regularly for failed releases

## Summary

Troubleshooting Helm deployments in Rancher requires a systematic approach: check the release status, identify the specific failure scenario, diagnose using kubectl and Helm CLI commands, and apply the appropriate fix. Most failures fall into common categories like image pull errors, resource constraints, PVC issues, and template rendering problems. Rancher provides visibility into release status through its UI, while the Helm CLI and kubectl give you the tools to dig deeper into the root cause. Use preventive measures like dry runs, atomic upgrades, and staging environments to minimize production failures.
