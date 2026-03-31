# How to Configure Promotion Thresholds for Cache Tiering in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, CacheTiering, Storage, Performance

Description: Learn how to configure object promotion thresholds in Ceph cache tiering to control when objects are moved from the backing pool to the cache pool based on access frequency.

---

Promotion thresholds in Ceph cache tiering control how aggressively objects are moved from the slow backing pool to the fast cache pool. Setting these thresholds correctly prevents cache thrashing and ensures only truly hot data occupies the limited cache space.

## Promotion Decision Factors

Ceph evaluates two primary signals before promoting an object:

1. **Hit set recency**: How many recent time windows contain this object (controlled by `min_read_recency_for_promote`)
2. **Cache fullness**: Whether the cache has room for new objects (controlled by `cache_target_full_ratio`)

Both conditions must be favorable for promotion to occur.

## Recency-Based Promotion Threshold

```bash
# Objects must appear in at least N recent hit set windows to be promoted
ceph osd pool set cache-pool min_read_recency_for_promote 2

# Same threshold for write-triggered promotions (writeback mode)
ceph osd pool set cache-pool min_write_recency_for_promote 2
```

Lower values (1) make the cache more aggressive - any recently accessed object is promoted. Higher values (3-4) make the cache conservative, only promoting objects with sustained access patterns.

## Checking Promotion Rate

After configuring thresholds, monitor how many objects are being promoted per second:

```bash
ceph osd pool stats cache-pool
```

```text
cache tier io 150 MiB/s promote, 30 MiB/s evict, 50 MiB/s flush
```

If the promote rate is too high, increase `min_read_recency_for_promote`. If it is too low and cache utilization is low despite a warm workload, decrease the threshold.

## Cache Full Ratio and Promotion Blocking

When the cache pool is above `cache_target_full_ratio`, promotions are blocked until eviction frees space:

```bash
# Block promotions when cache exceeds 80% full
ceph osd pool set cache-pool cache_target_full_ratio 0.8

# Emergency: block all I/O when cache exceeds 95% full
ceph osd pool set cache-pool cache_max_evict_check_size 10
```

## Target Max Bytes (Promotion Ceiling)

Setting `target_max_bytes` gives Ceph a capacity target for the cache pool, which influences flush and eviction scheduling:

```bash
# Set cache pool capacity to 200 GiB
ceph osd pool set cache-pool target_max_bytes 214748364800
```

When the cache approaches this limit, the flush and eviction daemons run more aggressively to make room for new promotions.

## Tuning Promotion Aggressiveness

```text
Scenario              min_read_recency   cache_target_full_ratio
Hot dataset fits      1-2                0.9 (allow high utilization)
Mixed hot/cold        2-3                0.8 (balanced)
Mostly sequential     3-4                0.7 (conservative, anti-scan)
Write-heavy           2                  0.7 (leave room for dirty objects)
```

## Target Object Count

You can also limit promotion by object count rather than bytes:

```bash
# Allow maximum 1 million objects in cache pool
ceph osd pool set cache-pool target_max_objects 1000000
```

When the object count exceeds this threshold, eviction begins regardless of byte capacity.

## Viewing All Cache Pool Settings

```bash
ceph osd pool get cache-pool all | grep -E "cache|hit_set|min_|target"
```

```text
min_read_recency_for_promote: 2
min_write_recency_for_promote: 2
cache_target_dirty_ratio: 0.4
cache_target_full_ratio: 0.8
target_max_bytes: 214748364800
hit_set_count: 12
hit_set_period: 3600
```

## Summary

Promotion thresholds control the flow of objects from the slow backing pool to the fast cache pool. Use `min_read_recency_for_promote` to filter out single-access objects and `cache_target_full_ratio` to ensure the cache does not overfill and block new promotions. Monitor promote and evict rates to validate that your thresholds are working as intended for your specific workload pattern.
