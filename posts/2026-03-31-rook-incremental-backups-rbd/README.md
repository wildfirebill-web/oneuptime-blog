# How to Configure Incremental Backups with RBD

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Backup, Incremental, Snapshot, Storage

Description: Set up incremental backups for Ceph RBD images using export-diff to transfer only changed blocks between snapshots, reducing backup time and storage consumption.

---

## Overview

Ceph RBD's `export-diff` command exports only the blocks that changed between two snapshots. This enables highly efficient incremental backups that transfer a fraction of the data compared to full backups, without sacrificing restore flexibility.

## How RBD Incremental Backups Work

The workflow is:
1. Take a full export on day 0
2. Create a new snapshot daily
3. Export only the diff between the previous and current snapshot
4. Retain a chain of diffs for restore

## Step 1 - Take Initial Full Backup

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

# Create baseline snapshot
rbd snap create replicapool/myvolume@snap-base

# Export full image
rbd export replicapool/myvolume@snap-base /backup/myvolume-base.img

echo "Base snapshot size: $(ls -lh /backup/myvolume-base.img)"
```

## Step 2 - Take Daily Incremental Backup

```bash
PREV_SNAP=snap-base
TODAY_SNAP=snap-$(date +%Y-%m-%d)

# Create new snapshot
rbd snap create replicapool/myvolume@${TODAY_SNAP}

# Export diff from previous snapshot
rbd export-diff \
  --from-snap ${PREV_SNAP} \
  replicapool/myvolume@${TODAY_SNAP} \
  /backup/myvolume-diff-${TODAY_SNAP}.img

echo "Incremental backup size: $(ls -lh /backup/myvolume-diff-${TODAY_SNAP}.img)"
```

## Step 3 - Inspect Diff Contents

```bash
# View which blocks changed in the diff
rbd export-diff \
  --from-snap snap-base \
  replicapool/myvolume@snap-2026-03-31 - | \
  rbd merge-diff - /dev/null
```

## Step 4 - Restore from Incremental Chain

To restore, import the base image then apply each diff in order:

```bash
# Import base backup to a new image
rbd import /backup/myvolume-base.img replicapool/myvolume-restored

# Apply incremental diffs in order
rbd import-diff /backup/myvolume-diff-snap-2026-03-29.img replicapool/myvolume-restored
rbd import-diff /backup/myvolume-diff-snap-2026-03-30.img replicapool/myvolume-restored
rbd import-diff /backup/myvolume-diff-snap-2026-03-31.img replicapool/myvolume-restored
```

## Step 5 - Automate with a Script

```bash
#!/bin/bash
POOL=replicapool
IMAGE=myvolume
BACKUP_DIR=/backup
DATE=$(date +%Y-%m-%d)
PREV_DATE=$(date -d yesterday +%Y-%m-%d)

NEW_SNAP="snap-${DATE}"
PREV_SNAP="snap-${PREV_DATE}"

kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  rbd snap create ${POOL}/${IMAGE}@${NEW_SNAP}

kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  rbd export-diff \
  --from-snap ${PREV_SNAP} \
  ${POOL}/${IMAGE}@${NEW_SNAP} - > ${BACKUP_DIR}/diff-${DATE}.img

# Remove old snapshots beyond 30 days
OLD_SNAP="snap-$(date -d '-30 days' +%Y-%m-%d)"
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  rbd snap rm ${POOL}/${IMAGE}@${OLD_SNAP} 2>/dev/null || true

echo "Incremental backup completed: ${BACKUP_DIR}/diff-${DATE}.img"
```

## Step 6 - Monitor Backup Chain Integrity

```bash
# List all snapshots for the image
rbd snap ls replicapool/myvolume

# Verify diff file integrity
file /backup/diff-2026-03-31.img
```

## Summary

RBD incremental backups using `export-diff` dramatically reduce both backup window time and storage requirements by transferring only changed blocks. The key is maintaining a consistent snapshot chain - never remove an intermediate snapshot that is referenced by a diff in your backup chain. For weekly full backups combined with daily incrementals, the storage savings are typically 80-95% compared to daily full backups.
