# How to Remove Namespaces in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Namespaces, DevOps

Description: Learn how to safely remove Kubernetes namespaces in Portainer, including handling stuck deletions and backing up important resources.

## Introduction

Removing a Kubernetes namespace in Portainer deletes ALL resources contained within it — pods, services, deployments, PVCs, secrets, and more. This is a destructive operation that requires careful planning. This guide covers the safe namespace removal process.

## Prerequisites

- Portainer with Kubernetes environment
- Admin access
- A namespace to remove (not a system namespace)

## Step 1: Inventory Namespace Resources

Before deleting, list all resources to understand what will be lost:

```bash
# Comprehensive resource list
kubectl get all -n old-namespace

# Storage (PVCs contain data!)
kubectl get pvc -n old-namespace

# ConfigMaps and Secrets
kubectl get configmap,secret -n old-namespace

# Ingresses
kubectl get ingress -n old-namespace

# Custom resources
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -I{} kubectl get {} -n old-namespace 2>/dev/null
```

## Step 2: Back Up Important Data

```bash
# Export all resources as YAML backup
kubectl get all -n old-namespace -o yaml > old-namespace-backup.yaml

# Backup secrets (sensitive data)
kubectl get secrets -n old-namespace -o yaml > old-namespace-secrets.yaml

# Backup PVC data using a temporary pod
kubectl run backup \
  --image=alpine \
  --namespace=old-namespace \
  --overrides='{"spec":{"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"my-data-pvc"}}],"containers":[{"name":"backup","image":"alpine","command":["tar","czf","/backup/data.tar.gz","/data"],"volumeMounts":[{"name":"data","mountPath":"/data"}]}]}}' \
  --restart=Never
```

## Step 3: Remove the Namespace via Portainer

1. Select your Kubernetes environment
2. Click **Namespaces** in the sidebar
3. Find the namespace you want to remove
4. Click the **Remove** button (trash icon)
5. Confirm in the dialog

**Warning:** This deletes ALL resources including PersistentVolumeClaims (and potentially the underlying data, depending on the reclaim policy).

## Step 4: Remove via kubectl

```bash
# Delete the namespace
kubectl delete namespace old-namespace

# Watch deletion progress
kubectl get namespace old-namespace -w

# Status progression:
# Active → Terminating → (removed)
```

## Step 5: Handle Stuck Terminating Namespace

Sometimes namespaces get stuck in `Terminating` state due to resources with finalizers:

```bash
# Check what's preventing deletion
kubectl get namespace old-namespace -o json | jq '.spec.finalizers'
kubectl get namespace old-namespace -o json | jq '.status'

# List remaining resources in the namespace
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -n 1 kubectl get --ignore-not-found -n old-namespace 2>/dev/null
```

### Fix Stuck Namespace

```bash
# Method 1: Remove finalizers via kubectl proxy
kubectl proxy &
PROXY_PID=$!

# Get the namespace JSON
kubectl get namespace old-namespace -o json > /tmp/ns.json

# Remove finalizers
cat /tmp/ns.json | jq '.spec.finalizers = []' > /tmp/ns-clean.json

# Apply via the proxy
curl -X PUT http://localhost:8001/api/v1/namespaces/old-namespace/finalize \
  -H "Content-Type: application/json" \
  --data-binary @/tmp/ns-clean.json

kill $PROXY_PID
```

```bash
# Method 2: Direct patch (safer)
kubectl patch namespace old-namespace \
  --type=json \
  -p='[{"op": "remove", "path": "/spec/finalizers"}]'
```

### Find Resources with Finalizers

```bash
# Find all resources with finalizers in the namespace
kubectl get all -n old-namespace -o json | \
  jq -r '.items[] | select(.metadata.finalizers != null) |
  "\(.kind)/\(.metadata.name): \(.metadata.finalizers)"'

# Remove finalizer from a specific resource
kubectl patch customresource my-resource -n old-namespace \
  --type=json \
  -p='[{"op": "remove", "path": "/metadata/finalizers"}]'
```

## Step 6: Clean Up Cluster-Scoped Resources

Some resources associated with the namespace are cluster-scoped and persist:

```bash
# PersistentVolumes (if reclaim policy is Retain)
kubectl get pv | grep old-namespace

# ClusterRoles and ClusterRoleBindings created for the namespace
kubectl get clusterrolebinding | grep old-namespace
kubectl delete clusterrolebinding portainer-rw-old-namespace

# External DNS entries (if using external-dns)
# These should auto-delete, but verify
```

## Step 7: Verify Complete Deletion

```bash
# Confirm namespace is gone
kubectl get namespace old-namespace
# Error: namespace "old-namespace" not found

# Verify no PVs remain with Released status
kubectl get pv | grep old-namespace

# Check for remaining cluster-scoped resources
kubectl get clusterrolebinding | grep old-namespace
kubectl get rolebinding -A | grep old-namespace
```

## Safely Removing Shared Namespaces (Production)

For production namespaces, follow this process:

```bash
# 1. Announce planned removal with advance notice
# 2. Set the namespace to block new deployments
kubectl annotate namespace production \
  scheduler.alpha.kubernetes.io/node-selector="environment=decommissioned"

# 3. Scale down all deployments gradually
kubectl get deployments -n production -o name | \
  xargs kubectl scale --replicas=0 -n production

# 4. Migrate data from PVCs
# 5. Update DNS and remove external dependencies
# 6. After final verification, delete the namespace
kubectl delete namespace production
```

## Conclusion

Namespace deletion is straightforward but irreversible. Always inventory resources, back up important data (especially PVC contents), and handle any stuck finalizers before assuming the deletion failed. For production namespace decommissioning, follow a structured process with advance notice and data migration to ensure nothing important is lost.
