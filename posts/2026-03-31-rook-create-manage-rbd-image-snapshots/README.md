# How to Create and Manage RBD Image Snapshots

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Snapshot, Data Protection, Kubernetes

Description: Learn how to create, list, and manage RADOS Block Device image snapshots in Ceph, including how to use VolumeSnapshots in Kubernetes via the Rook CSI driver.

---

## RBD Snapshot Overview

RADOS Block Device (RBD) snapshots capture the state of a block device image at a specific point in time. RBD snapshots are:
- **Instant** - created in milliseconds
- **Copy-on-write** - only changed blocks are stored after the snapshot
- **Efficient** - multiple snapshots share unchanged data

## Creating an RBD Snapshot via CLI

```bash
# List existing images
rbd ls mypool

# Create a snapshot
rbd snap create mypool/myimage@snap-20260331

# List snapshots for an image
rbd snap ls mypool/myimage
```

Output:
```
SNAPID  NAME            SIZE    PROTECTED  TIMESTAMP
1       snap-20260331   10 GiB  no         Tue Mar 31 10:00:00 2026
```

## Creating a Snapshot via Kubernetes VolumeSnapshot

First, create a VolumeSnapshotClass:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-rbdplugin-snapclass
driver: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  csi.storage.k8s.io/volumesnapshot/name: $(volumesnapshot.name)
  csi.storage.k8s.io/volumesnapshot/namespace: $(volumesnapshot.namespace)
  csi.storage.k8s.io/volumesnapshotcontent/name: $(volumesnapshotcontent.name)
deletionPolicy: Delete
```

Then create a VolumeSnapshot:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-pvc-snapshot
  namespace: default
spec:
  volumeSnapshotClassName: csi-rbdplugin-snapclass
  source:
    persistentVolumeClaimName: my-pvc
```

Apply:

```bash
kubectl apply -f snapshot.yaml
kubectl get volumesnapshot my-pvc-snapshot
```

## Restoring a PVC from a Snapshot

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc-restored
spec:
  storageClassName: rook-ceph-block
  dataSource:
    name: my-pvc-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl apply -f restored-pvc.yaml
```

## Managing Snapshot Space Usage

Snapshots consume space as the original data changes. Check snapshot space usage:

```bash
rbd du mypool/myimage
```

Output:
```
NAME            PROVISIONED  USED
myimage@snap-1  10 GiB       1.2 GiB
myimage@snap-2  10 GiB       800 MiB
myimage         10 GiB       3.1 GiB
```

## Deleting Snapshots

```bash
# Delete a specific snapshot
rbd snap rm mypool/myimage@snap-20260331

# Delete all snapshots for an image
rbd snap purge mypool/myimage
```

In Kubernetes:

```bash
kubectl delete volumesnapshot my-pvc-snapshot
```

## Protecting Snapshots

Mark a snapshot as protected to prevent accidental deletion:

```bash
rbd snap protect mypool/myimage@snap-20260331
```

Unprotect before deleting:

```bash
rbd snap unprotect mypool/myimage@snap-20260331
rbd snap rm mypool/myimage@snap-20260331
```

## Summary

RBD image snapshots are instant, copy-on-write point-in-time copies of block device images. Create them via `rbd snap create` for direct Ceph access, or via Kubernetes VolumeSnapshot objects for CSI-managed PVCs. Restore snapshots by creating new PVCs with the snapshot as the `dataSource`. Monitor snapshot space usage with `rbd du` and use snapshot protection to guard against accidental deletion of important recovery points.
