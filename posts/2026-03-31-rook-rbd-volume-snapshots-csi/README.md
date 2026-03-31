# How to Create RBD Volume Snapshots with Rook CSI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Snapshot, Storage, Kubernetes

Description: Learn how to create RBD volume snapshots using Rook CSI VolumeSnapshotClass and VolumeSnapshot resources in Kubernetes.

---

## Overview

Rook CSI supports Kubernetes volume snapshots for RBD (RADOS Block Device) volumes. Snapshots allow you to capture a point-in-time copy of a PersistentVolumeClaim, which can be used for backup purposes or to provision new volumes from a known-good state.

## Prerequisites

- Kubernetes 1.20 or later
- The `snapshot.storage.k8s.io` CRDs installed (VolumeSnapshot, VolumeSnapshotClass, VolumeSnapshotContent)
- Rook 1.9 or later
- The CSI snapshotter sidecar running in the provisioner deployment

Verify the snapshot CRDs are present:

```bash
kubectl get crd | grep volumesnapshot
```

## Create a VolumeSnapshotClass

The VolumeSnapshotClass defines the driver and deletion policy for snapshots:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-rbdplugin-snapclass
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  csi.storage.k8s.io/snapshotter-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/snapshotter-secret-namespace: rook-ceph
deletionPolicy: Delete
```

Apply it:

```bash
kubectl apply -f rbd-snapshotclass.yaml
```

## Create a VolumeSnapshot

Once the VolumeSnapshotClass is ready, take a snapshot of an existing PVC:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: rbd-pvc-snapshot
spec:
  volumeSnapshotClassName: csi-rbdplugin-snapclass
  source:
    persistentVolumeClaimName: rbd-pvc
```

```bash
kubectl apply -f rbd-snapshot.yaml
kubectl get volumesnapshot rbd-pvc-snapshot
```

## Verify the Snapshot is Ready

Check the snapshot status:

```bash
kubectl describe volumesnapshot rbd-pvc-snapshot
```

The output should show `readyToUse: true` and reference a `VolumeSnapshotContent` object. You can also inspect the content directly:

```bash
kubectl get volumesnapshotcontent
kubectl describe volumesnapshotcontent <content-name>
```

## Provision a New Volume from the Snapshot

To create a new PVC from the snapshot, set the `dataSource` field to reference the snapshot:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-pvc-from-snapshot
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

The new PVC will be pre-populated with the data from the snapshot.

## Delete a Snapshot

To remove a snapshot (and its underlying RBD snapshot if the deletion policy is `Delete`):

```bash
kubectl delete volumesnapshot rbd-pvc-snapshot
```

## Summary

Rook CSI enables Kubernetes-native RBD volume snapshots via the `VolumeSnapshotClass` and `VolumeSnapshot` APIs. By defining a snapshot class that references the Rook RBD CSI driver, you can capture point-in-time copies of block volumes and restore them into new PVCs - making backup and recovery workflows straightforward within the Kubernetes ecosystem.
