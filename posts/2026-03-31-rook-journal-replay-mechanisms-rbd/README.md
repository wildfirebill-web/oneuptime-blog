# How to Understand Journal Replay Mechanisms in RBD

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Journal, Mirroring, Internals

Description: Learn how RBD journal replay works in Ceph, including how writes are recorded, replayed on secondary clusters, and how to troubleshoot replay issues.

---

## Overview

Journal replay is the mechanism by which the `rbd-mirror` daemon on the secondary cluster reads journal entries from the primary cluster and applies them to the local copy of the image. Understanding this process helps you tune performance, diagnose replication lag, and troubleshoot errors effectively.

## How the RBD Journal Works

When journaling is enabled on an RBD image:

1. Every write is first recorded to the journal object(s) in the same RADOS pool
2. The write is acknowledged to the client after the journal entry is committed
3. The image object is updated asynchronously from the journal
4. The `rbd-mirror` daemon on the secondary reads these journal entries over the network

Journal objects are stored as `rbd_journal.<image-id>.*` in the pool.

```bash
# List journal objects for an image
IMAGE_ID=$(rbd info replicapool/myimage --format json | jq -r '.block_name_prefix' | sed 's/rbd_data\.//')
rados -p replicapool ls | grep "rbd_journal.$IMAGE_ID"
```

## Viewing Journal Details

```bash
# Show journal information
rbd journal info replicapool/myimage

# Show journal status including commit position
rbd journal status replicapool/myimage

# Example output includes:
# registered clients:
#   client.1: commit_position=[...], object_set=0, entry_num=1234
```

## Understanding Journal Positions

The journal tracks positions for:
- **commit_position** - last journal entry applied to the image
- **mirror_position** - last entry replicated to secondary

The difference between `master_position` (primary's latest) and `mirror_position` (secondary's progress) represents replication lag.

```bash
# Check lag on the secondary
rbd mirror image status replicapool/myimage --format json | \
  jq '.description'
```

## Tuning Journal Replay Performance

```bash
# Increase replay threads for faster catch-up
ceph config set rbd-mirror rbd_journal_max_concurrent_object_sets 32

# Adjust the polling age for lower latency
ceph config set rbd-mirror rbd_mirror_journal_poll_age 2

# Set commit age - lower means less lag but more IOPS
rbd config image set replicapool/myimage rbd_journal_commit_age 1
```

## Troubleshooting Journal Replay Issues

Common issues and diagnostics:

```bash
# Check rbd-mirror daemon logs for replay errors
sudo journalctl -u ceph-rbd-mirror@rbd-mirror.0 | grep -i "error\|replay"

# In Rook, check mirror daemon pod logs
kubectl -n rook-ceph logs -l app=rook-ceph-rbd-mirror --tail=100

# Check for journal overflow (journal too small for write rate)
rbd journal info replicapool/myimage | grep "overflow"
```

If the journal is overflowing, increase journal object size:

```bash
# Journal order 24 = 16MB per object
rbd config image set replicapool/myimage rbd_journal_order 24
```

## Journal Pruning

Committed journal entries are pruned automatically. If the journal grows large, check that the mirror daemon is making progress:

```bash
# Verify commit position is advancing
watch "rbd journal status replicapool/myimage | grep commit"
```

If commit position is stuck, restart the mirror daemon:

```bash
sudo systemctl restart ceph-rbd-mirror@rbd-mirror.0
```

## Summary

RBD journal replay is the core mechanism of journal-based mirroring. Writes are first committed to the journal, then the rbd-mirror daemon on the secondary reads and replays these entries. Monitor lag using `entries_behind_master`, tune replay performance with concurrent object set settings, and diagnose issues by checking daemon logs and journal status. Proper journal sizing prevents overflow that can stall replication.
