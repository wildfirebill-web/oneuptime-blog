# How to Create RBD Snapshots

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Snapshot, Backup

Description: Learn how to create, manage, and restore RBD image snapshots in Rook-Ceph using both the rbd CLI and Kubernetes VolumeSnapshot API.

---

## RBD Snapshot Overview

RBD snapshots are point-in-time, read-only copies of an RBD image. They are stored efficiently using copy-on-write (COW) semantics - only data that changes after the snapshot is stored separately. Snapshots are the foundation for RBD cloning and mirroring.

In Rook-Ceph, snapshots can be managed either through the Kubernetes VolumeSnapshot API or directly via the `rbd` CLI using the Rook toolbox.

## Creating a Snapshot via CLI

Access the Rook toolbox:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- bash
```

Create a snapshot of an RBD image:

```bash
rbd snap create replicapool/myimage@snap-2026-03-31
```

List snapshots for an image:

```bash
rbd snap ls replicapool/myimage
```

Sample output:

```text
SNAPID  NAME              SIZE    PROTECTED  TIMESTAMP
4       snap-2026-03-31   10 GiB  no         Tue Mar 31 10:00:00 2026
```

## Protecting and Unprotecting Snapshots

Before cloning from a snapshot, protect it to prevent deletion:

```bash
rbd snap protect replicapool/myimage@snap-2026-03-31
rbd snap unprotect replicapool/myimage@snap-2026-03-31
```

## Rolling Back to a Snapshot

Rollback an image to a previous snapshot state:

```bash
rbd snap rollback replicapool/myimage@snap-2026-03-31
```

Note: This overwrites all data written since the snapshot was taken.

## Using Kubernetes VolumeSnapshot API

Create a VolumeSnapshot for a PVC backed by RBD:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: rbd-pvc-snapshot
  namespace: default
spec:
  volumeSnapshotClassName: csi-rbdplugin-snapclass
  source:
    persistentVolumeClaimName: my-rbd-pvc
```

Check snapshot status:

```bash
kubectl get volumesnapshot rbd-pvc-snapshot -n default
```

## Restoring a Snapshot to a New PVC

Create a PVC from the snapshot:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-restored-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: rook-ceph-block
  dataSource:
    name: rbd-pvc-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

## Deleting Snapshots

Remove a snapshot via CLI:

```bash
rbd snap rm replicapool/myimage@snap-2026-03-31
```

Remove all snapshots from an image:

```bash
rbd snap purge replicapool/myimage
```

## Summary

RBD snapshots in Rook-Ceph provide efficient point-in-time copies of block volumes using COW semantics. You can manage them via the `rbd` CLI for direct operations or through the Kubernetes VolumeSnapshot API for Kubernetes-native workflows. Protect snapshots before cloning, and use the VolumeSnapshot restore path to provision new PVCs from snapshots for application recovery scenarios.
