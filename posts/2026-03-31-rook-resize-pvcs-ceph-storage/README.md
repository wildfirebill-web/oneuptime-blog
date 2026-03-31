# How to Resize PVCs Backed by Ceph Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, PVC, Resize, Expand, Storage, CSI

Description: Expand Kubernetes PersistentVolumeClaims backed by Ceph RBD or CephFS online without downtime using the CSI volume expansion feature.

---

## Online Volume Expansion with Ceph

Ceph CSI supports online volume expansion for both RBD (block) and CephFS volumes. You can grow a PVC while it is mounted and in use - no Pod restart required for most cases. The StorageClass must have `allowVolumeExpansion: true`.

## Verifying Expansion is Enabled

```bash
# Check StorageClass allows expansion
kubectl get storageclass rook-ceph-block -o jsonpath='{.allowVolumeExpansion}'
# Should return: true

# If not enabled, patch the StorageClass
kubectl patch storageclass rook-ceph-block \
  -p '{"allowVolumeExpansion": true}'
```

## Resizing an RBD PVC

Edit the PVC's storage request to the new size:

```bash
# Method 1: kubectl edit
kubectl edit pvc db-data -n production
# Change spec.resources.requests.storage from 100Gi to 200Gi

# Method 2: kubectl patch
kubectl patch pvc db-data -n production \
  -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'
```

You can only increase the storage size - shrinking is not supported.

## Verifying the Expansion

```bash
# Watch the PVC for resize completion
kubectl get pvc db-data -n production -w

# Look for the condition
kubectl describe pvc db-data -n production | grep -A5 Conditions

# Check the actual filesystem size inside the pod
kubectl exec -n production deploy/my-app -- df -h /data
```

The expansion happens in two phases:
1. The Ceph RBD image is resized (fast, seconds)
2. The filesystem inside the image is expanded (requires the volume to be mounted)

## Resizing a CephFS PVC

CephFS expansion follows the same process:

```bash
kubectl patch pvc shared-assets -n production \
  -p '{"spec":{"resources":{"requests":{"storage":"500Gi"}}}}'
```

CephFS expansion is quota-based and typically completes within seconds since there is no filesystem resize needed - CephFS manages quotas at the directory level.

## Handling Expansion Failures

If the PVC gets stuck in `FilesystemResizePending`:

```bash
# Check events on the PVC
kubectl describe pvc db-data -n production | grep -i event

# Check CSI node plugin logs
kubectl logs -n rook-ceph -l app=csi-rbdplugin -c csi-rbdplugin | tail -50

# Force re-trigger by restarting the Pod (mounts the volume fresh)
kubectl rollout restart deployment/my-app -n production
```

## Automating Resize with VPA or KEDA

For dynamic workloads, you can script automated PVC expansion:

```bash
#!/bin/bash
# Check if PVC is more than 80% full and expand
NAMESPACE="production"
PVC_NAME="db-data"
CURRENT_SIZE=$(kubectl get pvc $PVC_NAME -n $NAMESPACE -o jsonpath='{.spec.resources.requests.storage}')
NEW_SIZE="200Gi"  # Calculate based on growth rate

kubectl patch pvc $PVC_NAME -n $NAMESPACE \
  -p "{\"spec\":{\"resources\":{\"requests\":{\"storage\":\"${NEW_SIZE}\"}}}}"
```

## Summary

Ceph CSI makes online PVC expansion straightforward - edit the storage request in the PVC spec and Ceph handles resizing both the RBD image and the filesystem without interrupting running workloads. CephFS expansions are quota-based and complete almost instantly, while RBD expansions require the volume to be mounted for filesystem expansion to complete. Always verify the new size from inside the Pod rather than just checking the PVC object.
