# How to Use Ceph with Kubernetes VolumeSnapshot API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, VolumeSnapshot, CSI, Backup, RBD, CephFS

Description: Use the Kubernetes VolumeSnapshot API with Ceph CSI to create, manage, and restore point-in-time snapshots of PersistentVolumes backed by Ceph RBD and CephFS.

---

## Kubernetes VolumeSnapshot API Overview

The Kubernetes VolumeSnapshot API (stable since 1.20) provides a standardized way to create point-in-time snapshots of PersistentVolumes. Ceph CSI fully supports this API for both RBD and CephFS volumes, enabling consistent snapshot workflows regardless of the underlying storage type.

## Prerequisites

Install the snapshot CRDs and controller if not already present:

```bash
# Apply snapshot CRDs
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v8.0.0/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v8.0.0/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v8.0.0/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml

# Deploy the snapshot controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v8.0.0/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v8.0.0/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
```

## Creating a VolumeSnapshotClass for RBD

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-rbdplugin-snapclass
driver: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  csi.storage.k8s.io/snapshotter-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/snapshotter-secret-namespace: rook-ceph
deletionPolicy: Delete
```

## Taking a Snapshot

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: db-snapshot-2026-03-31
  namespace: production
spec:
  volumeSnapshotClassName: csi-rbdplugin-snapclass
  source:
    persistentVolumeClaimName: db-data
```

## Checking Snapshot Status

```bash
kubectl get volumesnapshot db-snapshot-2026-03-31 -n production
kubectl describe volumesnapshot db-snapshot-2026-03-31 -n production | grep -E "Ready|Size|Error"

# List all VolumeSnapshotContents (the actual snapshot objects)
kubectl get volumesnapshotcontent
```

## Restoring from a Snapshot

Create a new PVC that sources from the snapshot:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data-restored
  namespace: production
spec:
  storageClassName: rook-ceph-block
  dataSource:
    name: db-snapshot-2026-03-31
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

The restored PVC is a clone of the snapshot and is immediately writable. It does not affect the original PVC.

## CephFS VolumeSnapshotClass

For CephFS volumes, create a separate VolumeSnapshotClass:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-cephfsplugin-snapclass
driver: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  csi.storage.k8s.io/snapshotter-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/snapshotter-secret-namespace: rook-ceph
deletionPolicy: Delete
```

## Summary

The Kubernetes VolumeSnapshot API with Ceph CSI provides a consistent, declarative interface for point-in-time data protection on both RBD block volumes and CephFS filesystems. Snapshots are created quickly using Ceph's copy-on-write mechanism and can be restored into new PVCs within minutes, making them suitable for pre-upgrade checkpoints, data recovery, and testing environment seeding.
