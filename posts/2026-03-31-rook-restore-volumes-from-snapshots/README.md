# How to Restore Volumes from Snapshots with Rook CSI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Snapshot, Restore, Kubernetes

Description: Learn how to restore Kubernetes PersistentVolumes from RBD or CephFS snapshots using Rook CSI dataSource references in PVC definitions.

---

## Overview

Rook CSI supports restoring both RBD and CephFS volumes from previously created snapshots. The restore process uses the standard Kubernetes `dataSource` field in a PersistentVolumeClaim, making it easy to create new volumes pre-populated with snapshot data. This guide covers the restore workflow for both storage types.

## Prerequisites

Before restoring, confirm a snapshot exists and is ready:

```bash
kubectl get volumesnapshot
kubectl describe volumesnapshot <snapshot-name>
```

The `Status.ReadyToUse` field must be `true`. If it is `false`, the snapshot creation may still be in progress or may have failed.

## Restore an RBD Volume from Snapshot

To provision a new RBD PVC from a snapshot, reference the snapshot in the `dataSource` field:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-restored
  namespace: default
spec:
  storageClassName: rook-ceph-block
  dataSource:
    name: rbd-pvc-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

The storage size in the restore PVC must be greater than or equal to the original snapshot size.

## Restore a CephFS Volume from Snapshot

For CephFS, the process is identical - only the StorageClass and access mode differ:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-restored
  namespace: default
spec:
  storageClassName: rook-cephfs
  dataSource:
    name: cephfs-pvc-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```

## Monitor the Restore Progress

Apply the PVC and watch its status:

```bash
kubectl apply -f restored-pvc.yaml
kubectl get pvc rbd-restored -w
```

The PVC transitions from `Pending` to `Bound` once the CSI driver finishes cloning the snapshot data. For large volumes, this may take several minutes depending on the data size.

Check CSI provisioner logs for progress:

```bash
kubectl logs -n rook-ceph deployment/csi-rbdplugin-provisioner -c csi-provisioner | tail -20
```

## Use the Restored Volume in a Pod

Once the PVC is `Bound`, attach it to a pod and verify data integrity:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restore-test
spec:
  containers:
  - name: verify
    image: busybox
    command: ["ls", "-la", "/mnt/data"]
    volumeMounts:
    - mountPath: /mnt/data
      name: restored
  volumes:
  - name: restored
    persistentVolumeClaim:
      claimName: rbd-restored
```

```bash
kubectl apply -f restore-test.yaml
kubectl logs restore-test
```

## Cross-Namespace Restore

By default, a VolumeSnapshot can only be used as a dataSource within the same namespace. To restore across namespaces, you must create a `VolumeSnapshotContent` manually and bind it to a new `VolumeSnapshot` object in the target namespace:

```bash
kubectl get volumesnapshotcontent <content-name> -o yaml > snapshot-content.yaml
# Edit the snapshot content to reference a new VolumeSnapshot in the target namespace
kubectl apply -f snapshot-content.yaml -n target-namespace
```

## Summary

Restoring volumes from snapshots with Rook CSI is a native Kubernetes operation using the `dataSource` PVC field. Whether restoring RBD block volumes or CephFS shared filesystem volumes, the workflow is consistent and does not require manual Ceph commands. The restored PVC can be used immediately once it transitions to `Bound` status.
