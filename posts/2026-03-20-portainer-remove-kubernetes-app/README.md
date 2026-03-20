# How to Remove a Kubernetes Application in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Application, DevOps

Description: Learn how to safely remove Kubernetes applications and their associated resources in Portainer without leaving orphaned objects.

## Introduction

Removing a Kubernetes application in Portainer deletes the deployment and associated resources. However, some resources like PersistentVolumeClaims and Secrets may need to be removed separately. This guide covers the complete process of safely removing an application.

## Prerequisites

- Portainer with Kubernetes environment
- Application to remove
- Understanding of what resources need to be kept vs deleted

## Step 1: List Resources Before Deletion

Before removing, understand what the application has created:

```bash
# List all resources for an application

kubectl get all -n production -l app=my-app

# Output:
# NAME                          READY   STATUS
# pod/my-app-xxx-yyy            1/1     Running

# NAME              TYPE        CLUSTER-IP    PORT(S)
# service/my-app    ClusterIP   10.96.1.100   80/TCP

# NAME                     READY   UP-TO-DATE   AVAILABLE
# deployment.apps/my-app   3/3     3            3

# NAME                             DESIRED   CURRENT   READY
# replicaset.apps/my-app-abc123    3         3         3
```

## Step 2: Check for PVCs and Other Resources

```bash
# Check PVCs (will persist after app deletion by default)
kubectl get pvc -n production -l app=my-app

# Check ConfigMaps
kubectl get configmap -n production -l app=my-app

# Check Secrets
kubectl get secrets -n production -l app=my-app

# Check HPA
kubectl get hpa -n production -l app=my-app

# Check Ingress
kubectl get ingress -n production -l app=my-app
```

## Step 3: Remove the Application via Portainer

1. Navigate to **Applications** in the Kubernetes environment
2. Find the application in the list
3. Click the **Remove** button (trash icon)

Or open the application detail:
1. Click on the application name
2. Click **Remove this application**
3. In the confirmation dialog, choose what to keep:

```text
Remove options:
  [x] Delete deployment
  [x] Delete services
  [ ] Delete persistent volumes   (keep data by default)
  [ ] Delete ConfigMaps
  [ ] Delete Secrets
```

4. Click **Remove**

## Step 4: Delete Associated PVCs (If No Longer Needed)

PVCs are not deleted by default to protect your data:

```bash
# List PVCs related to the application
kubectl get pvc -n production

# Delete specific PVC (IRREVERSIBLE - data will be lost)
kubectl delete pvc my-app-data -n production

# Or in Portainer: navigate to Volumes and delete the PVC
```

**Warning:** Deleting a PVC permanently deletes the data if the reclaim policy is `Delete`. Always back up important data before deleting PVCs.

## Step 5: Clean Up Associated Resources

```bash
# Delete ConfigMaps
kubectl delete configmap app-config -n production

# Delete Secrets (be careful - other apps may use them)
kubectl delete secret app-secrets -n production

# Delete HPA
kubectl delete hpa my-app-hpa -n production

# Delete Ingress
kubectl delete ingress my-app-ingress -n production
```

## Step 6: Remove via YAML

If the application was deployed via YAML, delete using the same manifest:

```bash
# Delete all resources defined in a YAML file
kubectl delete -f my-app.yaml -n production

# This deletes exactly what the YAML created
```

## Step 7: Remove Helm-Deployed Applications

For Helm-based applications:

```bash
# List Helm releases
helm list --namespace production

# Uninstall a release (removes all chart resources)
helm uninstall my-app --namespace production

# Check for any remaining resources
kubectl get all -n production -l app.kubernetes.io/instance=my-app
```

In Portainer: navigate to the Helm releases section and click **Remove** on the release.

## Step 8: Verify Complete Removal

```bash
# Confirm pods are terminated
kubectl get pods -n production -l app=my-app

# Confirm services are removed
kubectl get svc -n production -l app=my-app

# Confirm no orphaned resources
kubectl get all -n production | grep my-app
```

## Cleanup Complete Application Namespace

If you want to remove everything in a namespace:

```bash
# Delete entire namespace (removes ALL resources in it)
kubectl delete namespace my-old-namespace

# WARNING: This deletes everything including PVCs and Secrets
```

In Portainer: navigate to **Namespaces** and use the delete option.

## Step 9: Handle Finalizers (Stuck Deletions)

Sometimes resources get stuck in a `Terminating` state due to finalizers:

```bash
# Check for finalizers
kubectl get pod stuck-pod -n production -o json | jq '.metadata.finalizers'

# Force remove by patching out the finalizer
kubectl patch pod stuck-pod -n production \
  --type json \
  --patch='[{"op": "remove", "path": "/metadata/finalizers"}]'

# For PVCs stuck terminating
kubectl patch pvc stuck-pvc -n production \
  --type json \
  --patch='[{"op": "remove", "path": "/metadata/finalizers"}]'
```

## Conclusion

Removing Kubernetes applications in Portainer is straightforward but requires careful consideration of associated resources. PVCs are intentionally preserved to protect data - always decide explicitly whether to delete them. For clean environments, check for orphaned resources after removal and consider using namespace-level cleanup when decommissioning entire environments. Use YAML or Helm for reproducible deployments that can be cleanly removed with the same manifest that created them.
