# How to Manage the CephFS Purge Queue

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, MDS, Storage

Description: Learn how the CephFS purge queue works and how to manage it to ensure deleted files are properly cleaned up from RADOS storage in Rook-Ceph deployments.

---

## Overview

When files are deleted in CephFS, the MDS does not immediately remove the underlying RADOS objects. Instead, it places the inode on a purge queue, which asynchronously deletes the RADOS objects in the background. This design keeps delete operations fast (the directory entry is removed instantly) while the actual data cleanup happens asynchronously. Understanding the purge queue is important when storage is not being reclaimed after deletes.

## How the Purge Queue Works

When a client deletes a file:

1. The MDS removes the directory entry immediately
2. If the file has no remaining hard links, the inode is placed on the purge queue
3. The MDS purge manager asynchronously deletes RADOS data objects
4. Once all RADOS objects are deleted, the inode is removed from the metadata pool

```text
Client delete -> MDS removes dentry -> Inode added to purge queue
                                           |
                                    Background: delete RADOS objects
                                           |
                                    Inode removed from metadata pool
```

## Monitor Purge Queue Status

Check the current purge queue depth and throughput:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph tell mds.cephfs:0 perf dump | jq '.purge_queue'
```

Key metrics to watch:

```text
pq_executing        - Number of inodes currently being purged
pq_executing_ops    - Number of RADOS delete operations in progress
pq_item_in_journal  - Purge items still in the journal (not yet queued)
```

## Check if Purge Queue is Backing Up

If storage is not being reclaimed after deletes, check if the purge queue is falling behind:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph tell mds.cephfs:0 dump_historic_ops | grep purge
```

Also check MDS logs for purge-related messages:

```bash
kubectl -n rook-ceph logs -l app=rook-ceph-mds,rook_file_system=cephfs \
  --tail=200 | grep -i purge
```

## Tune Purge Queue Concurrency

Control how many files the purge manager processes concurrently:

```bash
# Maximum number of files to purge in parallel
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_purge_queue_max_concurrent 32

# Maximum RADOS operations per purge batch
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_max_purge_ops 16

# Per-OSD limit for purge operations (prevents OSD overload)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_max_purge_ops_per_pg 0.5
```

## Force Purge Queue Flush

There is no direct "flush purge queue" command, but you can increase the urgency by temporarily raising concurrency limits and then restoring them:

```bash
# Temporarily increase purge concurrency
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_max_purge_ops 64

# Monitor progress
watch kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph tell mds.cephfs:0 perf dump \| jq '.purge_queue.pq_executing'

# Restore defaults
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_max_purge_ops 16
```

## Purge Queue and Snapshots

Files that are deleted but captured in a snapshot are NOT purged until all snapshots referencing the file are also deleted. This is why storage may not be reclaimed immediately after deleting files in directories with active snapshots.

```bash
# List snapshots that may be holding data
ls /mnt/cephfs/.snap/
rmdir /mnt/cephfs/.snap/old-snapshot  # delete old snapshots to free space
```

## Summary

The CephFS purge queue is the mechanism responsible for asynchronously cleaning up RADOS data objects after file deletion. If storage capacity is not decreasing after large deletes, check the purge queue depth with `ceph tell mds perf dump` and tune `mds_max_purge_ops` to increase cleanup throughput. Remember that files captured in snapshots will not be purged until all referencing snapshots are removed, which is a common cause of unexpected storage consumption in Rook-Ceph deployments.
