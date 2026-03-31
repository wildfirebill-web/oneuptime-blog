# How to Fix "backfill_toofull" Preventing Recovery in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Troubleshooting, Backfill, Recovery, OSD, Capacity

Description: Resolve the backfill_toofull condition in Ceph that blocks PG recovery by freeing space, rebalancing OSDs, or adjusting backfill thresholds.

---

## Introduction

The `backfill_toofull` state occurs when Ceph cannot backfill (recover) data to target OSDs because those OSDs are too full. This prevents the cluster from reaching a healthy state after OSD failures or rebalancing. This guide explains how to break out of this cycle.

## Identifying the Problem

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

Example:

```
HEALTH_WARN backfill_toofull; 8 pgs are stuck unclean for more than 300 sec
8 pgs backfill_toofull
```

Check which OSDs are near full:

```bash
ceph osd df sort -n
```

## Step 1 - Identify the Bottleneck OSD

```bash
ceph osd df
```

Look for OSDs with high usage percentage. The `VAR` column shows variance from average - high positive values indicate overcrowded OSDs.

## Step 2 - Raise the Backfill Full Ratio

This allows backfill to proceed on slightly fuller OSDs:

```bash
# Check current threshold
ceph config get osd osd_backfillfull_ratio

# Raise temporarily
ceph osd set-backfillfull-ratio 0.95
ceph osd set-nearfull-ratio 0.90
```

Monitor whether backfill resumes:

```bash
watch "ceph -s | grep -E 'backfill|recovery'"
```

## Step 3 - Reweight Overcrowded OSDs

Reduce the weight of full OSDs to discourage further data placement:

```bash
# Reduce weight of a full OSD
ceph osd reweight osd.3 0.7

# Or use the automatic reweighter
ceph osd reweight-by-utilization 80
```

This triggers rebalancing away from full OSDs.

## Step 4 - Add More OSDs

If the cluster is globally near full, add capacity:

```bash
# In CephCluster spec, add more nodes/disks
storage:
  nodes:
  - name: "worker-5"
    devices:
    - name: "sdb"
    - name: "sdc"
```

## Step 5 - Delete Unnecessary Data

If adding capacity is not immediately possible, free space:

```bash
# Find and delete old snapshots
rbd ls replicapool | while read vol; do
  rbd snap ls replicapool/$vol
done

# Delete a specific snapshot
rbd snap rm replicapool/myvol@old-snapshot

# Check bucket usage in RGW
radosgw-admin bucket stats | python3 -m json.tool | grep -E "bucket|size"
```

## Step 6 - Enable pg_upmap for Better Balancing

For persistent imbalance, enable the `balancer` module:

```bash
ceph mgr module enable balancer
ceph balancer mode upmap
ceph balancer on
```

Check the balancer's plan:

```bash
ceph balancer status
ceph balancer eval
```

## Summary

The `backfill_toofull` condition prevents PG recovery when target OSDs are too full. The fix involves temporarily raising backfill ratio thresholds, reweighting overcrowded OSDs to trigger rebalancing, adding new OSD capacity, or deleting unnecessary data. Enabling the Ceph balancer module helps maintain even distribution across OSDs to prevent this condition from recurring.
