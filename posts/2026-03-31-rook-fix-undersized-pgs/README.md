# How to Fix Undersized PGs in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Placement Group, Replication, Troubleshooting

Description: Learn how to diagnose and fix undersized Placement Groups in Ceph, which indicate insufficient replicas and reduced data redundancy.

---

## What Are Undersized PGs?

A Placement Group is "undersized" when its acting set contains fewer OSDs than the pool's `size` (replication factor). For example, in a pool with `size=3`, a PG with only 2 active replicas is undersized. The data is still accessible (above `min_size=2`), but redundancy is reduced. If another OSD fails before recovery completes, data could become unavailable.

## Identifying Undersized PGs

Check for undersized PGs in cluster health:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph health detail
```

Look for `PG_DEGRADED` or `HEALTH_WARN n pgs undersized`.

Count undersized PGs:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg stat
```

List specific undersized PGs:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg dump_stuck undersized
```

## Common Causes

1. **An OSD is down**: The most common cause. Ceph marks PGs undersized until the OSD returns or recovery completes.
2. **OSD marked out**: If an OSD is `out`, its PGs begin backfill to other OSDs. During this process, some PGs are temporarily undersized.
3. **Insufficient OSDs for pool size**: If your pool has `size=3` but only 2 OSDs exist, all PGs will be permanently undersized.

## Fixing OSD-Related Undersized PGs

First, identify which OSD is causing the undersized state:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd tree | grep down
```

If the OSD is temporarily down (pod restarting), wait for recovery:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-osd
```

If the OSD pod is stuck, restart it:

```bash
kubectl -n rook-ceph delete pod rook-ceph-osd-<id>-<suffix>
```

## Checking Recovery Progress

Monitor recovery as Ceph brings PGs back to full replication:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  watch -n 5 "ceph status | grep -E 'pgs|degraded|undersized'"
```

## Fixing Structural Undersized Issues

If your pool `size` exceeds the number of available OSDs, either add OSDs:

```bash
# Add storage nodes to the CephCluster spec
kubectl -n rook-ceph edit cephcluster rook-ceph
```

Or reduce the pool `size` to match available OSDs:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set replicapool size 2

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set replicapool min_size 1
```

## Preventing Undersized PGs

Set `noout` during planned maintenance to prevent Ceph from marking OSDs out:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd set noout
```

Unset after maintenance:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd unset noout
```

## Summary

Undersized PGs indicate insufficient replicas and reduced data redundancy. They are usually caused by a down OSD and resolve automatically once the OSD recovers or backfill completes. For structural issues where pool `size` exceeds OSD count, either add OSDs or reduce the pool replication factor. Use `noout` during maintenance to prevent unnecessary undersized PG events.
