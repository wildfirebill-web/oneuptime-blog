# How to Configure Promotion Thresholds for Cache Tiering

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Cache Tiering, Promotion, Performance Tuning

Description: Configure Ceph cache tier promotion thresholds to control which objects are elevated to the fast cache layer based on read and write access frequency.

---

## What Is Cache Tier Promotion

In Ceph cache tiering, promotion is the process of moving an object from the slow backing pool to the fast cache pool when the object becomes "hot" (frequently accessed). The promotion thresholds control how aggressively objects are promoted.

Setting thresholds too low means almost every object gets promoted, overwhelming the cache. Setting them too high means popular objects stay on slow storage longer than necessary.

## Promotion Threshold Parameters

### min_read_recency_for_promote

The minimum number of recent hit sets in which an object must appear before it is promoted on a read request:

```bash
# Require object to be seen in at least 2 of the recent hit sets
ceph osd pool set mycache min_read_recency_for_promote 2
```

With `hit_set_count=4` and this set to 2, an object must be accessed in 2 out of the last 4 time windows before being promoted.

### min_write_recency_for_promote

Same concept but for write requests:

```bash
# Require object to be in 2 recent hit sets before promoting on write
ceph osd pool set mycache min_write_recency_for_promote 2
```

### target_max_bytes and target_max_objects

These control when the cache starts evicting objects (not strictly promotion, but related):

```bash
# Cache pool size limits (triggers flushing/eviction when reached)
ceph osd pool set mycache target_max_bytes 107374182400  # 100 GiB
ceph osd pool set mycache target_max_objects 1000000
```

## Configuring a Complete Cache Tier

```bash
# Step 1: Create pools
ceph osd pool create slow-pool 128 128 replicated
ceph osd pool create fast-cache 64 64 replicated

# Step 2: Configure tiering relationship
ceph osd tier add slow-pool fast-cache
ceph osd tier cache-mode fast-cache writeback
ceph osd tier set-overlay slow-pool fast-cache

# Step 3: Configure hit sets for tracking
ceph osd pool set fast-cache hit_set_type bloom
ceph osd pool set fast-cache hit_set_count 6
ceph osd pool set fast-cache hit_set_period 1800  # 30 minutes

# Step 4: Set promotion thresholds
ceph osd pool set fast-cache min_read_recency_for_promote 2
ceph osd pool set fast-cache min_write_recency_for_promote 2

# Step 5: Set cache size limits
ceph osd pool set fast-cache target_max_bytes 53687091200  # 50 GiB
```

## Tuning Promotion Aggressiveness

Different workloads benefit from different promotion strategies:

### Aggressive Promotion (hot streaming data)

```bash
# Promote after being seen in just 1 hit set
ceph osd pool set fast-cache min_read_recency_for_promote 1
ceph osd pool set fast-cache min_write_recency_for_promote 1
ceph osd pool set fast-cache hit_set_period 300  # 5 minute windows
```

### Conservative Promotion (steady-state datasets)

```bash
# Require 3 of 6 hit set windows
ceph osd pool set fast-cache hit_set_count 6
ceph osd pool set fast-cache hit_set_period 3600  # 1 hour windows
ceph osd pool set fast-cache min_read_recency_for_promote 3
ceph osd pool set fast-cache min_write_recency_for_promote 3
```

## Immediate Promotion Mode

For workloads where all reads should be cached immediately (no access pattern tracking):

```bash
# Set to 0 to promote on every access
ceph osd pool set fast-cache min_read_recency_for_promote 0
```

This is equivalent to a simple cache: every read that misses is promoted immediately.

## Monitoring Cache Effectiveness

```bash
# View cache hit rates
ceph osd pool stats fast-cache

# Sample output fields to watch:
# hit_set_stats: how many objects are being tracked
# promote_op: number of promotions happening

# Watch cache fills and evictions
watch -n 5 ceph df detail
```

## Understanding Eviction vs Promotion Balance

If the cache is filling too fast (rapid promotion, slow eviction), increase thresholds:

```bash
# More selective promotion
ceph osd pool set fast-cache min_read_recency_for_promote 3

# More aggressive eviction
ceph osd pool set fast-cache cache_target_dirty_ratio 0.2
ceph osd pool set fast-cache cache_target_full_ratio 0.7
```

If the cache is underutilized (too little promotion), lower thresholds:

```bash
ceph osd pool set fast-cache min_read_recency_for_promote 1
```

## Read vs Write Promotion Asymmetry

In some workloads, writes are always hot (every write should be cached), but reads follow a Pareto distribution:

```bash
# Promote all writes immediately, reads need 2 hit sets
ceph osd pool set fast-cache min_write_recency_for_promote 0
ceph osd pool set fast-cache min_read_recency_for_promote 2
```

## Summary

Promotion thresholds in Ceph cache tiering control how frequently an object must be accessed before it is elevated to the fast cache pool. The `min_read_recency_for_promote` and `min_write_recency_for_promote` parameters specify how many recent hit set windows the object must appear in. Setting these to 0 promotes everything immediately, while higher values (2-4) provide more selective promotion based on actual access patterns. Balance promotion aggressiveness with cache capacity and eviction settings for optimal performance.
