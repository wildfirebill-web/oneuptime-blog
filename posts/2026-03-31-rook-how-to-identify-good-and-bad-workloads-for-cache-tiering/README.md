# How to Identify Good and Bad Workloads for Cache Tiering

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Cache Tiering, Workload Analysis, Performance

Description: Identify which workloads benefit from Ceph cache tiering and which are harmed by it, with guidance on measuring workload characteristics before deployment.

---

## Why Workload Fit Matters

Cache tiering is not universally beneficial. The wrong workload can actually perform worse with caching than without it, due to promotion/eviction overhead, flush latency, and the extra I/O path complexity. Understanding your workload's characteristics is essential before deploying cache tiering.

Note: Cache tiering is deprecated in Ceph Pacific. This guidance applies to existing deployments or legacy evaluation contexts.

## Characteristics of Cache-Friendly Workloads

A workload benefits from cache tiering when it has these properties:

### Temporal Locality

The same objects are accessed repeatedly over short time windows. This allows the cache to serve repeated reads from fast media.

```text
Good: A web application serving the same product images repeatedly
Bad: A backup job that reads each file exactly once
```

### Working Set Fits in Cache

The "hot" portion of data (frequently accessed objects) must fit in the cache pool capacity. If the working set is larger than the cache, every access will be a cache miss.

```text
Good: 10% of objects get 90% of reads, and that 10% fits in cache
Bad: All objects are accessed uniformly - no hot subset
```

### Read-Heavy Access Pattern

Cache tiering provides the most benefit for reads. Write-heavy workloads encounter extra overhead from writeback flush cycles.

```text
Good: 80% reads, 20% writes (typical web app serving static assets)
Bad: 90% writes (log aggregation, time-series ingestion)
```

### Large Object Access

Promotion is most efficient when objects are large (the promotion cost is amortized across many reads of the same large object):

```text
Good: Video files, VM disk images accessed frequently
Bad: Tiny objects (small keys in a KV store)
```

## Characteristics of Cache-Unfriendly Workloads

### Sequential Scan Workloads

Backup and analytics jobs read each object once in sequence, causing:
- Constant cache misses (no locality)
- Cache thrashing (promoting then immediately evicting)
- Backing pool gets same I/O as without cache

```bash
# Example: large sequential read benchmark
rados bench -p mypool 60 seq -t 1 -b 4M
# Cache tiering adds overhead here, not benefit
```

### Write-Dominated Workloads

Workloads that mostly write new objects:

```text
Bad fit: Log aggregation (always new objects)
Bad fit: Database WAL (sequential appends to new regions)
Bad fit: Time-series metrics storage (always new time buckets)
```

### Random Write with Small Objects

Small random writes require frequent flush operations and produce many dirty cache entries:

```text
Bad fit: OLTP databases with frequent small updates
Bad fit: RBD volumes for busy MySQL servers
```

### Already-Fast Backing Storage

If the backing pool already uses SSDs, adding another SSD cache layer provides no benefit:

```bash
# Check backing pool device class
ceph osd pool get mypool crush_rule
ceph osd crush rule dump <rule>
# If already using ssd device class, cache tiering adds no value
```

## Measuring Workload Characteristics

Use rados bench to characterize a workload:

```bash
# Simulate random access pattern
rados bench -p mypool 60 rand -t 8 -b 4K --no-cleanup

# Compare with sequential
rados bench -p mypool 60 seq -t 8 -b 4M --no-cleanup

# Mixed read/write (custom scripts or fio)
fio --filename=/dev/rbd/pool/vol --rw=randrw --rwmixread=70 \
  --bs=4k --iodepth=32 --runtime=60 --name=test
```

## Decision Framework

```text
Question                              | Answer  | Cache Tier Fit
Is the working set smaller than cache?| Yes     | Good
Is access pattern temporal/local?     | Yes     | Good
Is workload read-heavy (>60% reads)?  | Yes     | Good
Are objects large (>1 MB each)?       | Yes     | Good
Is workload sequential scan?          | Yes     | Poor
Is backing pool already on SSD?       | Yes     | Poor
Is workload write-heavy (>60% writes)?| Yes     | Poor
Are objects tiny (<4 KB)?             | Yes     | Poor
```

## Better Alternatives for Poor-Fit Workloads

For workloads that do not fit cache tiering:

```bash
# For sequential workloads - separate SSD pool for hot data
ceph osd pool create hot-pool 64 64 replicated
ceph osd pool set hot-pool crush_rule ssd_rule
# Move hot dataset to this pool explicitly

# For write-heavy workloads - BlueStore WAL on NVMe
# Configure in Rook CephCluster or cephadm OSD spec

# For small objects - tune BlueStore cache directly
ceph config set osd bluestore_cache_size 4294967296  # 4 GiB
```

## Summary

Good workloads for Ceph cache tiering have strong temporal locality (same objects accessed repeatedly), a hot working set that fits in the cache, and are read-dominated. Poor candidates include sequential scan workloads (backups, analytics), write-heavy workloads, small random write patterns, and workloads where the backing storage is already fast. Since cache tiering is deprecated in Ceph Pacific, even well-fitting workloads should be migrated to explicit SSD pools or BlueStore tuning in new deployments.
