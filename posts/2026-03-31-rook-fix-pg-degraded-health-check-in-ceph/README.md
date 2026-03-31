# How to Fix PG_DEGRADED Health Check in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Placement Group, Health Check, Replication

Description: Learn how to resolve PG_DEGRADED in Ceph, a warning indicating placement groups have fewer replicas than configured, usually due to OSD failures or rebalancing.

---

## What Is PG_DEGRADED?

`PG_DEGRADED` is a Ceph health warning that means one or more Placement Groups have fewer object replicas than their pool's configured `size`. While the PGs are still accessible (unlike `PG_AVAILABILITY`), the cluster is operating with reduced redundancy. If another OSD fails before recovery completes, data could become unavailable or lost.

This state is common during normal cluster operations - OSD replacements, rolling reboots, or adding new nodes all temporarily cause PG degradation.

## Identifying Degraded PGs

Check cluster status:

```bash
ceph -s
ceph health detail
```

Example output:

```text
[WRN] PG_DEGRADED: Degraded data redundancy: 142/426 objects degraded (33.333%), 10 pgs degraded
    pg 2.1 is active+degraded objects degraded
```

Get details on specific PGs:

```bash
ceph pg dump_stuck degraded
ceph pg <pg-id> query
```

Find which OSDs are down or out:

```bash
ceph osd stat
ceph osd tree
```

## Common Causes and Fixes

### Cause 1 - OSD is Down Temporarily

If an OSD crashed or the node was rebooted, recovery begins automatically. Monitor it:

```bash
watch ceph -s
```

If an OSD pod keeps crashing in Rook, check logs:

```bash
kubectl -n rook-ceph logs -l app=rook-ceph-osd,ceph-osd-id=3
```

Restart if needed:

```bash
kubectl -n rook-ceph rollout restart deploy/rook-ceph-osd-3
```

### Cause 2 - Not Enough OSDs for the Replication Factor

If your pool `size` is 3 but you only have 2 OSDs, PGs will always be degraded:

```bash
ceph osd pool get <pool-name> size
ceph osd stat
```

Either add more OSDs or reduce the pool size:

```bash
ceph osd pool set <pool-name> size 2
ceph osd pool set <pool-name> min_size 1
```

### Cause 3 - Recovery is In Progress

Degradation during recovery is normal. Speed up recovery by increasing recovery threads:

```bash
ceph config set osd osd_recovery_max_active 5
ceph config set osd osd_recovery_op_priority 63
ceph config set osd osd_max_backfills 5
```

To throttle recovery so it doesn't impact client I/O:

```bash
ceph config set osd osd_recovery_max_active 1
```

### Cause 4 - Cluster is Rebalancing After Adding OSDs

After adding OSDs, the cluster backfills data to redistribute. This is normal and will resolve automatically. Monitor:

```bash
ceph pg stat
```

## Checking Recovery Progress

```bash
ceph -s | grep recovering
ceph osd perf
```

## Summary

`PG_DEGRADED` indicates placement groups have fewer replicas than desired. While data is still accessible, redundancy is reduced. Fix it by recovering down OSDs, ensuring adequate OSD count for pool replication factor, and tuning recovery parameters to balance speed versus client I/O impact. The condition clears automatically once all replicas are restored.
