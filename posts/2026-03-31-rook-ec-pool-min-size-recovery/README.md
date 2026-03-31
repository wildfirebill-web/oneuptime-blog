# How to Configure min_size for Erasure Coded Pool Recovery in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, ErasureCoding, Recovery, Storage

Description: Learn how to configure the min_size parameter for erasure coded pools in Ceph to control write availability during OSD failures and recovery operations.

---

The `min_size` parameter on a Ceph pool defines the minimum number of replicas (or EC shards) that must be available for writes to proceed. For erasure coded pools, this setting controls whether the cluster allows writes during degraded states when some OSD shards are unavailable.

## Understanding size and min_size for EC Pools

For an EC pool with k data chunks and m coding chunks:

- `size` = k + m (total shards per object)
- `min_size` = minimum shards available for writes to proceed (default: k + 1)

The default `min_size` of `k + 1` means writes proceed as long as at least one parity shard is available, providing some protection even during degraded states. Setting `min_size = k` allows writes with no parity redundancy, which risks data loss if another OSD fails during recovery.

## Checking Current Pool Settings

```bash
ceph osd pool get ec-pool size
ceph osd pool get ec-pool min_size
```

For a k=4, m=2 profile:

```text
size: 6
min_size: 5
```

## Setting min_size

To be more conservative (only write when 2 parity shards are available):

```bash
ceph osd pool set ec-pool min_size 6
```

This requires all 6 shards to be available, meaning any OSD failure blocks writes. Use this for critical data where availability is secondary to durability.

To allow writes with minimal redundancy:

```bash
ceph osd pool set ec-pool min_size 5
```

This is the default for k=4, m=2 - writes proceed if at least 5 of 6 shards are available (1 OSD failure tolerated for writes).

**Warning:** Never set `min_size` below `k` for EC pools. With fewer than k shards, Ceph cannot even reconstruct existing data, let alone accept new writes safely.

## min_size During Recovery

During OSD recovery, some shards may temporarily be unavailable. If `min_size` is too high, writes are blocked until recovery completes. If it is too low, writes may succeed without full parity, creating a window of reduced durability.

Monitor degraded shards during recovery:

```bash
ceph health detail
```

```text
HEALTH_WARN 1 osds down, 1 pg degraded
pg 8.0 is degraded (1/6 objects degraded) recovery 2 active+degraded
```

## Rook CephBlockPool Configuration

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ec-pool
  namespace: rook-ceph
spec:
  erasureCoded:
    dataChunks: 4
    codingChunks: 2
  parameters:
    min_size: "5"    # Allow writes with 1 OSD failure (conservative default)
```

## Impact on Cluster Behavior

```text
min_size Setting   OSD Failures Tolerated   Write Availability
k+m (6 for k=4,m=2)  0                     Blocked on any OSD failure
k+1 (5 for k=4,m=2)  1                     Writes proceed with 1 failure
k   (4 for k=4,m=2)  2 (risky)             Writes proceed with 2 failures, no parity
```

## Relationship with CRUSH Failure Domains

`min_size` works alongside CRUSH failure domains. If `crush_failure_domain=host` and a host goes down, all shards on that host are unavailable simultaneously. With k=4, m=2 and min_size=5, losing one host (which may hold one shard) still allows writes to continue.

## Summary

Set `min_size` to at least `k + 1` for EC pools to ensure at least one parity shard is always available during writes in degraded states. Use `min_size = k + m` only when maximum durability is required and you can tolerate write blocking during any OSD failure. Never set `min_size` below `k` as this creates conditions where Ceph cannot guarantee data integrity.
