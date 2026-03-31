# How to Understand CephFS Snapshots Internals

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Snapshot, Storage

Description: Learn how CephFS snapshots work internally, including how they are stored in RADOS, how copy-on-write is implemented, and how the snapshot directory interface works.

---

## Overview

CephFS snapshots provide point-in-time copies of directory trees. Unlike block-level snapshots in RBD, CephFS snapshots operate at the filesystem level and are managed by the MDS. Understanding how snapshots are implemented internally helps you manage them correctly, understand their performance characteristics, and troubleshoot snapshot-related issues in Rook-Ceph deployments.

## Snapshot Architecture

CephFS snapshots are:
- Stored as hidden subdirectories named `.snap` within any directory
- Implemented using copy-on-write (COW) semantics at the RADOS object level
- Tracked by the MDS through snapshot sequence numbers (snapids)
- Independent per-subtree - you can snapshot any directory, not just the filesystem root

## How Copy-On-Write Works

When a snapshot is taken, the MDS records a snapshot sequence ID (snapid). Each RADOS object tracks which snapids it was created under using RADOS's native snapshot support. When a file is modified after a snapshot, the OSD writes new RADOS objects for the modified data while preserving the original objects for the snapshot.

```text
Time 0: file.dat -> RADOS objects [A, B, C]
          |
          snapshot "snap1" taken (snapid=42)
          |
Time 1: file.dat modified -> RADOS objects [A, B', C]
          Snapshot "snap1" still points to [A, B, C]
```

## Access Snapshots via the .snap Directory

The `.snap` directory is the user-facing interface to snapshots:

```bash
# List existing snapshots
ls /mnt/cephfs/.snap/

# Access a snapshot by name
ls /mnt/cephfs/.snap/snap1/

# Recover a file from a snapshot
cp /mnt/cephfs/.snap/snap1/important.txt /mnt/cephfs/recovered.txt
```

## Create and Delete Snapshots Manually

```bash
# Create a snapshot of the root directory
mkdir /mnt/cephfs/.snap/mysnap

# Delete a snapshot
rmdir /mnt/cephfs/.snap/mysnap
```

## Create Snapshots in Rook via CephFS CSI

Rook uses the VolumeSnapshot API for PVC-backed snapshot creation:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: cephfs-pvc-snapshot
spec:
  volumeSnapshotClassName: csi-cephfsplugin-snapclass
  source:
    persistentVolumeClaimName: my-cephfs-pvc
```

## List Snapshots via CLI

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs subvolume snapshot ls cephfs mysubvolume --group_name mygroup
```

## Snapshot Overhead

CephFS snapshots are lightweight at creation time (no data is copied). Storage overhead grows over time as data is modified after the snapshot. Monitor snapshot space usage:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph df detail | grep -A2 "cephfs-data"
```

## Snapshot Limits

Configure maximum snapshots per directory:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_max_snaps_per_dir 100
```

## Summary

CephFS snapshots are implemented as hidden `.snap` directories backed by RADOS copy-on-write semantics managed through MDS snapshot sequence IDs. They are lightweight at creation, transparent to access via the `.snap` directory interface, and can be taken on any directory subtree. In Rook-Ceph, snapshots integrate with the Kubernetes VolumeSnapshot API through the CephFS CSI driver, enabling application-consistent point-in-time backups of CephFS-backed PVCs.
