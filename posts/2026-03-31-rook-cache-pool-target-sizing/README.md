# How to Set Target Sizing for Cache Pools in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, CacheTiering, Capacity, Storage

Description: Learn how to configure target_max_bytes and target_max_objects for Ceph cache pools to control capacity limits and drive flush and eviction scheduling.

---

Cache pool sizing in Ceph is controlled by two primary parameters: `target_max_bytes` (maximum byte capacity) and `target_max_objects` (maximum object count). These settings tell Ceph how large the cache pool is intended to be, enabling the flush and eviction system to proactively manage cache utilization.

## Why Cache Pool Sizing Matters

Without size targets, Ceph's cache tier daemon does not know when to start flushing dirty objects or evicting clean objects. The cache pool would fill up completely before any proactive management begins, causing write stalls when the pool reaches 100% capacity.

Setting `target_max_bytes` allows Ceph to begin flushing at `cache_target_dirty_ratio` and evicting at `cache_target_full_ratio` of the configured target.

## Setting Target Maximum Bytes

```bash
# Set the cache pool target to 500 GiB
ceph osd pool set cache-pool target_max_bytes 536870912000

# Verify
ceph osd pool get cache-pool target_max_bytes
```

Choose this value based on the actual usable capacity of your SSD pool. Setting it too high means flush/eviction starts too late; too low means the cache is underutilized.

## Setting Target Maximum Objects

For workloads with many small objects, bytes alone may not be the right signal. Set an object count limit instead:

```bash
# Set object count target to 500,000 objects
ceph osd pool set cache-pool target_max_objects 500000

# Verify
ceph osd pool get cache-pool target_max_objects
```

If both `target_max_bytes` and `target_max_objects` are set, whichever threshold is reached first triggers flush and eviction.

## Understanding the Relationship with Ratio Settings

The ratio settings (`cache_target_dirty_ratio`, `cache_target_full_ratio`) are applied as fractions of the target:

```text
target_max_bytes = 500 GiB
cache_target_dirty_ratio = 0.4   --> Flush starts at 200 GiB dirty
cache_target_full_ratio = 0.8    --> Eviction starts at 400 GiB total
```

```bash
ceph osd pool set cache-pool cache_target_dirty_ratio 0.4
ceph osd pool set cache-pool cache_target_full_ratio 0.8
```

## Monitoring Cache Utilization

```bash
ceph df | grep cache-pool
```

```text
POOL         ID   STORED   OBJECTS   USED    %USED   MAX AVAIL
cache-pool   2    320 GiB   80000   480 GiB   64%    250 GiB
```

If `%USED` is consistently above 80%, your cache is too small for the working set. Consider increasing cache pool size or raising `min_read_recency_for_promote` to reduce promotion aggressiveness.

## Calculating Appropriate Cache Size

A general rule of thumb: the cache should be large enough to hold the active working set. If your workload has a hot set of 100 GiB that is accessed repeatedly, a 120-150 GiB cache target is reasonable (with some headroom for dirty ratio thresholds).

```text
hot_set_size = estimated frequently accessed data
cache_target = hot_set_size / cache_target_full_ratio
```

For example:
```text
hot_set = 100 GiB
cache_target_full_ratio = 0.8
cache_target = 100 / 0.8 = 125 GiB (set target_max_bytes to 134217728000)
```

## Example: Complete Cache Pool Sizing

```bash
ceph osd pool set cache-pool target_max_bytes 134217728000   # 125 GiB
ceph osd pool set cache-pool cache_target_dirty_ratio 0.4
ceph osd pool set cache-pool cache_target_full_ratio 0.8
ceph osd pool set cache-pool target_max_objects 1000000
```

## Summary

`target_max_bytes` and `target_max_objects` define the intended capacity of the cache pool and drive proactive flush and eviction scheduling. Size the cache to hold your hot working set plus headroom for dirty data and the full ratio threshold. Monitor actual cache utilization to validate that your sizing matches your workload's active dataset.
