# How to Remove a Kubernetes Application in Portainer - Application

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Application Management, Cleanup, DevOps

Description: Learn how to safely remove a Kubernetes application and its associated resources through the Portainer UI.

## Overview

Removing a Kubernetes application in Portainer deletes the Deployment (or other workload type) and optionally its associated resources like Services, ConfigMaps, Secrets, and PVCs.

## Removing an Application in Portainer

1. Select your Kubernetes environment.
2. Go to **Applications** in the sidebar.
3. Find the application to remove.
4. Either:
   - Click the checkbox next to the app and click **Remove** in the toolbar.
   - Or open the application and click the **Delete** button.
5. Confirm the deletion in the dialog.

## What Gets Deleted

When you remove an application in Portainer:

- The **Deployment**, **StatefulSet**, or **DaemonSet** resource is deleted.
- All **ReplicaSets** and **Pods** managed by it are terminated.
- **Services** associated with the deployment are optionally removed.

**Note**: Persistent Volume Claims (PVCs) and Secrets are **not** automatically deleted to prevent data loss. You must remove them separately.

## Removing Associated Resources via CLI

```bash
# Delete a deployment and its pods

kubectl delete deployment my-app --namespace=production

# Delete the associated service
kubectl delete service my-app --namespace=production

# Delete ConfigMaps
kubectl delete configmap app-config --namespace=production

# Delete Secrets (carefully!)
kubectl delete secret app-secrets --namespace=production

# Delete PVC (this destroys persistent data!)
kubectl delete pvc my-app-data-pvc --namespace=production
```

## Removing All Resources for an Application (Using Labels)

If all resources were created with consistent labels, delete them all at once:

```bash
# Delete all resources with a specific app label
kubectl delete all --selector=app=my-app --namespace=production

# Also delete PVCs with the same label
kubectl delete pvc --selector=app=my-app --namespace=production
```

## Removing from a Manifest

```bash
# Remove everything defined in a YAML manifest
kubectl delete -f my-app-deployment.yaml

# Or with a directory of manifests
kubectl delete -f ./manifests/
```

## Verifying Removal

```bash
# Confirm no pods remain
kubectl get pods --namespace=production | grep my-app

# Check for any leftover resources
kubectl get all --selector=app=my-app --namespace=production

# Verify PVCs are also removed if intended
kubectl get pvc --namespace=production
```

## Graceful Termination

Kubernetes sends a `SIGTERM` to containers before forceful termination. Ensure your application handles it:

```bash
# Check the termination grace period (default 30s)
kubectl get deployment my-app --namespace=production \
  -o jsonpath='{.spec.template.spec.terminationGracePeriodSeconds}'

# Force-delete a stuck pod (last resort)
kubectl delete pod stuck-pod --namespace=production --grace-period=0 --force
```

## Conclusion

Removing applications in Portainer is straightforward, but always verify that associated persistent data (PVCs) is intentionally preserved or deleted. Use label selectors via CLI to clean up all associated resources in one command.
