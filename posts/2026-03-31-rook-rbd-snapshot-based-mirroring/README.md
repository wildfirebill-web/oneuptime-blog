# How to Configure RBD Snapshot-Based Mirroring in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Mirroring, Snapshot

Description: Learn how to configure RBD snapshot-based mirroring in Rook to replicate block volumes between clusters on a scheduled basis without requiring the journaling feature.

---

## Overview

RBD snapshot-based mirroring transfers incremental snapshots of block volumes from a primary to a secondary Ceph cluster on a schedule. Unlike journal-based mirroring, snapshot mode does not require the `journaling` RBD feature, making it compatible with a wider range of kernel versions and image configurations. This guide covers configuring snapshot-based mirroring in Rook.

## How Snapshot Mirroring Works

Snapshot mirroring uses scheduled snapshots on the primary cluster and transfers only the changed blocks since the last snapshot to the secondary. The secondary maintains the latest consistent replica corresponding to the most recently transferred snapshot. Recovery point objective (RPO) is determined by the snapshot interval.

## Step 1 - Enable Pool-Level Mirroring in Snapshot Mode

Configure the CephBlockPool on the primary cluster for snapshot mirroring:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
  mirroring:
    enabled: true
    mode: image
    snapshotSchedules:
    - interval: 24h
      startTime: "00:00:00"
```

```bash
kubectl apply -f replicapool-snapshot-mirror.yaml
```

## Step 2 - Enable Snapshot Mirroring on Individual Images

Enable mirroring per image in snapshot mode:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror image enable replicapool/<image-name> snapshot
```

Snapshot mode does not require the `journaling` feature. Verify the image features:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd info replicapool/<image-name> | grep features
```

## Step 3 - Configure Snapshot Schedules

Add a snapshot schedule to the pool or individual images:

```bash
# Per-pool schedule
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror snapshot schedule add --pool replicapool 1h

# Per-image schedule
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror snapshot schedule add --pool replicapool \
  --image <image-name> 6h 00:00:00-05:00
```

List active schedules:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror snapshot schedule list --pool replicapool
```

## Step 4 - Exchange Bootstrap Tokens

Like journal mode, snapshot mirroring requires peer authentication between clusters:

```bash
# On primary cluster
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror pool peer bootstrap create \
  --site-name primary replicapool > bootstrap-token.txt

# On secondary cluster
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror pool peer bootstrap import \
  --site-name secondary \
  --direction rx-only \
  replicapool - < bootstrap-token.txt
```

## Step 5 - Verify Snapshot Mirroring Status

Check mirroring status and snapshot transfer progress:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror image status replicapool/<image-name>

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror pool status replicapool --verbose
```

A healthy snapshot mirror shows:

```text
state: up+stopped (on primary, when no active journal)
state: up+replaying (on secondary, when syncing)
local snapshot timestamp: <timestamp>
remote snapshot timestamp: <timestamp>
```

## Comparing Journal vs Snapshot Mirroring

```text
Feature           Journal Mode          Snapshot Mode
Kernel Req.       4.9+ (journaling)     Any modern kernel
Write Overhead    Higher (journal I/O)  Lower
RPO               Near-continuous       Snapshot interval
Feature Flag      journaling required   Not required
Crash Consistency Per-write ordering    Per-snapshot
```

## Summary

RBD snapshot-based mirroring in Rook provides a lower-overhead alternative to journal mirroring by transferring incremental snapshots on a schedule. It does not require the journaling image feature, making it suitable for images that cannot use journaling. Configure snapshot schedules based on your RPO requirements - shorter intervals improve recovery granularity at the cost of more frequent cross-cluster transfers.
