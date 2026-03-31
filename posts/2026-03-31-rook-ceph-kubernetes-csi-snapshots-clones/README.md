# How to Use Ceph with Kubernetes CSI Snapshots and Clones

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, CSI, Snapshot, Clone, Storage

Description: Use Rook-Ceph CSI drivers to create volume snapshots and clones in Kubernetes for backup, testing, and rapid provisioning workflows.

---

## Introduction

Kubernetes CSI snapshots allow you to capture point-in-time copies of PersistentVolumes, while clones enable instant copies for testing or staging environments. Rook-Ceph's CSI driver supports both operations for RBD and CephFS volumes.

## Prerequisites

- Rook-Ceph operator installed with CSI drivers
- `snapshot.storage.k8s.io` CRDs installed (VolumeSnapshotClass, VolumeSnapshot, etc.)
- A StorageClass backed by Rook-Ceph

Install snapshot CRDs if not present:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
```

## Creating a VolumeSnapshotClass

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-rbdplugin-snapclass
driver: rook-ceph.rbd.csi.ceph.com
deletionPolicy: Delete
parameters:
  clusterID: rook-ceph
  csi.storage.k8s.io/snapshotter-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/snapshotter-secret-namespace: rook-ceph
```

## Taking a Snapshot

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: myapp-snapshot
  namespace: default
spec:
  volumeSnapshotClassName: csi-rbdplugin-snapclass
  source:
    persistentVolumeClaimName: myapp-pvc
```

```bash
kubectl apply -f snapshot.yaml
kubectl get volumesnapshot myapp-snapshot
```

Wait for `READYTOUSE` to be `true`.

## Restoring from a Snapshot

Create a new PVC from the snapshot:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-restored
  namespace: default
spec:
  storageClassName: rook-ceph-block
  dataSource:
    name: myapp-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## Creating a Clone

Clones create an instant copy from an existing PVC without going through a snapshot:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-clone
  namespace: default
spec:
  storageClassName: rook-ceph-block
  dataSource:
    name: myapp-pvc
    kind: PersistentVolumeClaim
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Clones are useful for spinning up isolated test environments that mirror production data.

## Verifying Snapshots in Ceph

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd snap ls replicapool/csi-vol-abc123
```

## Summary

Rook-Ceph's CSI driver enables native Kubernetes volume snapshots and clones for both RBD and CephFS volumes. Snapshots provide point-in-time backups that can be restored as new PVCs, while clones offer near-instant data copies for development and testing - all managed declaratively through standard Kubernetes resources.
