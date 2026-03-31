# How to Use CephFS Snapshots for File-Level Backup

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Snapshot, Backup, File System, CSI

Description: Learn how to create and manage CephFS snapshots for file-level point-in-time backups, including Kubernetes VolumeSnapshot integration and individual file recovery.

---

## Overview

CephFS snapshots allow you to capture the state of a CephFS directory tree at a specific point in time. Unlike block-level snapshots, CephFS snapshots enable file-level recovery, making them ideal for workloads where you need to restore individual files rather than entire volumes.

## How CephFS Snapshots Work

CephFS snapshots are stored in a hidden `.snap` directory within each directory. When you create a snapshot, Ceph preserves the directory tree metadata and file data using copy-on-write, with virtually no upfront cost.

## Step 1 - Enable Snapshot Support in Rook

Verify snapshot support is enabled in your CephFilesystem:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - replicated:
        size: 3
  metadataServer:
    activeCount: 1
    activeStandby: true
```

## Step 2 - Create a VolumeSnapshotClass for CephFS

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-cephfsplugin-snapclass
driver: rook-ceph.cephfs.csi.ceph.com
deletionPolicy: Delete
parameters:
  clusterID: rook-ceph
  csi.storage.k8s.io/snapshotter-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/snapshotter-secret-namespace: rook-ceph
```

```bash
kubectl apply -f cephfs-snapclass.yaml
```

## Step 3 - Take a VolumeSnapshot

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: cephfs-snap-daily
  namespace: production
spec:
  volumeSnapshotClassName: csi-cephfsplugin-snapclass
  source:
    persistentVolumeClaimName: cephfs-pvc
```

```bash
kubectl apply -f cephfs-snapshot.yaml
kubectl get volumesnapshot -n production
```

## Step 4 - Access Files from the Snapshot

Mount a pod using the restored snapshot as a new PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-restored
  namespace: production
spec:
  storageClassName: rook-cephfs
  dataSource:
    name: cephfs-snap-daily
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
```

## Step 5 - Direct Snapshot Access via .snap Directory

If you have a CephFS volume mounted in a pod:

```bash
kubectl exec -it myapp-pod -- bash

# List snapshots
ls /data/.snap/

# Access files from a specific snapshot
ls /data/.snap/snap-2026-03-31/important-file.txt

# Copy a file from snapshot back to live directory
cp /data/.snap/snap-2026-03-31/lost-file.txt /data/
```

## Step 6 - Create a Snapshot Schedule

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystemSubVolumeGroup
metadata:
  name: myfs-group
  namespace: rook-ceph
spec:
  filesystemName: myfs
  pinning:
    distributed: 1
```

Use a CronJob for automated snapshots:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph fs snap-schedule add / --fs myfs 1d
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph fs snap-schedule add / --fs myfs 1w
```

## Summary

CephFS snapshots provide file-granularity point-in-time recovery that block-level snapshots cannot match. The Kubernetes VolumeSnapshot API integrates cleanly for application-level backups, while the `.snap` directory provides direct file access for quick individual file recovery. Pair automated snapshot schedules with a retention policy to maintain a rolling window of recovery points.
