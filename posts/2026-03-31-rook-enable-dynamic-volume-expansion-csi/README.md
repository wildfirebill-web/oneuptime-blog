# How to Enable Dynamic Volume Expansion with Rook CSI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, CSI, Storage

Description: Enable and use dynamic volume expansion with the Rook-Ceph CSI driver to resize PersistentVolumeClaims online without pod restarts.

---

## What Is Dynamic Volume Expansion

Dynamic volume expansion lets you resize a PersistentVolumeClaim (PVC) that is already bound and in use, without stopping the application. Rook's Ceph CSI driver supports online expansion for both RBD (block) and CephFS (filesystem) volumes.

The Kubernetes `VolumeExpansion` feature gate must be enabled (it is on by default since Kubernetes 1.24), and the StorageClass must declare `allowVolumeExpansion: true`.

## Enabling Volume Expansion in the StorageClass

If you are creating a new StorageClass, include the expansion flag:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

For an existing StorageClass, patch it:

```bash
kubectl patch storageclass rook-ceph-block \
  -p '{"allowVolumeExpansion": true}'
```

## Resizing a PVC

Edit the PVC and increase the `storage` request:

```bash
kubectl edit pvc my-pvc
```

Change the `spec.resources.requests.storage` value:

```yaml
spec:
  resources:
    requests:
      storage: 20Gi  # was 10Gi
```

Or use a patch command:

```bash
kubectl patch pvc my-pvc \
  -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

## Watching the Expansion

Kubernetes coordinates with the CSI driver to expand the volume. Monitor the PVC status:

```bash
kubectl describe pvc my-pvc
```

Look for the `Resizing` condition followed by `FileSystemResizePending` (for block volumes) or the final `Bound` status with the new size.

For CephFS volumes the resize is fully online. For RBD block volumes, the file system inside the volume is expanded when the pod is next restarted (or immediately if the node-level resize is triggered automatically).

## Verifying the Expanded Size

Inside the pod, check the file system:

```bash
kubectl exec -it <pod-name> -- df -h /data
```

For block devices, confirm the device size:

```bash
kubectl exec -it <pod-name> -- lsblk
```

On the Ceph side, inspect the image size:

```bash
kubectl -n rook-ceph exec -it <toolbox-pod> -- \
  rbd info replicapool/<pvc-image-name>
```

## Troubleshooting Expansion Issues

If the PVC is stuck in `FileSystemResizePending`:

```bash
# Check CSI node plugin logs
kubectl -n rook-ceph logs -l app=csi-rbdplugin -c csi-rbdplugin
```

Common causes:
- The pod using the PVC was not restarted after the block-level resize.
- The node plugin does not have permission to resize the file system.
- The StorageClass did not have `allowVolumeExpansion: true` at the time of provisioning.

## Summary

Dynamic volume expansion with Rook CSI requires setting `allowVolumeExpansion: true` on the StorageClass and then patching the PVC storage request. CephFS expansions complete online; RBD expansions need a pod restart to trigger the file system resize. Monitor the PVC conditions and CSI node plugin logs if the resize appears stuck.
