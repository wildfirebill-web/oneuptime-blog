# How to Remove Rook Finalizers from Stuck Resources

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Kubernetes, Finalizer, Cleanup, Troubleshooting

Description: Learn how to identify and remove Rook finalizers from Kubernetes resources stuck in Terminating state during Rook-Ceph cluster removal.

---

Rook uses Kubernetes finalizers to ensure proper cleanup ordering when resources are deleted. A finalizer on a resource prevents Kubernetes from actually deleting the object until the finalizer is removed. When the Rook operator is not running or is in a broken state, it cannot process and remove its own finalizers, leaving resources permanently stuck in `Terminating` state.

## Understanding Rook Finalizers

Rook adds finalizers to its custom resources to ensure Ceph daemons are cleaned up before the Kubernetes objects are deleted. Common Rook finalizers include:

```text
cephcluster.ceph.rook.io
cephblockpool.ceph.rook.io
cephfilesystem.ceph.rook.io
cephobjectstore.ceph.rook.io
```

These appear on the resource's `metadata.finalizers` field.

## Identifying Stuck Resources

List resources stuck in Terminating state:

```bash
kubectl -n rook-ceph get all
```

Resources stuck in terminating show no deletion timestamp progress. Check the deletion timestamp:

```bash
kubectl -n rook-ceph get cephcluster rook-ceph -o jsonpath='{.metadata.deletionTimestamp}'
```

If this shows a timestamp from long ago, the finalizer was never removed.

View all finalizers on a stuck resource:

```bash
kubectl -n rook-ceph get cephcluster rook-ceph \
  -o jsonpath='{.metadata.finalizers}'
```

```text
["cephcluster.ceph.rook.io"]
```

## Safe Removal: Restart the Operator First

Before manually removing finalizers, always try restarting the Rook operator to allow it to process them naturally:

```bash
kubectl -n rook-ceph rollout restart deployment rook-ceph-operator
kubectl -n rook-ceph rollout status deployment rook-ceph-operator
```

If the operator can connect to Ceph, it will remove the finalizers automatically after completing its cleanup logic. Wait a few minutes and check if the resource is gone.

## Manual Finalizer Removal

If the operator cannot process the finalizers (e.g., Ceph is completely down or the operator is failing to start), remove them manually.

### Method 1: kubectl patch

```bash
# Remove finalizers from CephCluster
kubectl -n rook-ceph patch cephcluster rook-ceph \
  --type merge \
  -p '{"metadata":{"finalizers":[]}}'
```

```bash
# Remove finalizers from CephBlockPool
kubectl -n rook-ceph patch cephblockpool replicapool \
  --type merge \
  -p '{"metadata":{"finalizers":[]}}'

# Remove finalizers from CephFilesystem
kubectl -n rook-ceph patch cephfilesystem myfs \
  --type merge \
  -p '{"metadata":{"finalizers":[]}}'
```

### Method 2: kubectl edit

```bash
kubectl -n rook-ceph edit cephcluster rook-ceph
```

Find and remove the `finalizers` section:

```yaml
metadata:
  finalizers:        # Delete this line
  - cephcluster.ceph.rook.io  # And this line
```

Save and exit.

### Method 3: Using the API Server Directly

For resources that `kubectl patch` cannot reach due to API errors:

```bash
kubectl -n rook-ceph get cephcluster rook-ceph -o json | \
  python3 -c "
import sys, json
d = json.load(sys.stdin)
d['metadata']['finalizers'] = []
print(json.dumps(d))
" | kubectl replace -f -
```

## Removing Finalizers from Multiple Resources

Script to remove finalizers from all Rook CRDs at once:

```bash
#!/bin/bash

NAMESPACE="rook-ceph"

echo "Removing finalizers from Rook custom resources..."

for resource_type in cephcluster cephblockpool cephfilesystem cephobjectstore \
  cephobjectstoreuser cephnfs cephfilesystemmirror; do

  RESOURCES=$(kubectl -n "$NAMESPACE" get "$resource_type" \
    -o jsonpath='{.items[*].metadata.name}' 2>/dev/null)

  for resource in $RESOURCES; do
    echo "Patching $resource_type/$resource"
    kubectl -n "$NAMESPACE" patch "$resource_type" "$resource" \
      --type merge \
      -p '{"metadata":{"finalizers":[]}}' 2>/dev/null || true
  done
done

echo "Done."
```

## Handling Stuck Namespace Termination

If the entire `rook-ceph` namespace is stuck in `Terminating` due to resources with finalizers, use the namespace finalize endpoint:

```bash
kubectl get namespace rook-ceph -o json | \
  python3 -c "
import sys, json
d = json.load(sys.stdin)
d['spec']['finalizers'] = []
print(json.dumps(d))
" | kubectl replace --raw /api/v1/namespaces/rook-ceph/finalize -f -
```

This directly patches the namespace spec via the API, bypassing the finalizer processing.

## Post-Removal Verification

After removing finalizers, verify the resources are actually deleted:

```bash
kubectl -n rook-ceph get cephcluster
kubectl -n rook-ceph get all
kubectl get namespace rook-ceph
```

Check that no orphaned resources remain:

```bash
kubectl get pv | grep rook
kubectl get crd | grep rook
```

## Summary

Rook finalizers prevent resources from being deleted until the operator processes them. When resources are stuck in `Terminating`, always try restarting the Rook operator first to allow natural cleanup. If the operator cannot process finalizers, remove them manually using `kubectl patch` with an empty finalizers array, or use the API server directly for namespace-level stuck terminations. After removal, verify all resources are deleted and check for orphaned PVs and CRDs.
