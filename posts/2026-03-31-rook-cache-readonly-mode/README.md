# How to Set Up Readonly Cache Mode in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, CacheTiering, Storage, Performance

Description: Learn how to configure readonly cache mode in Ceph, where hot objects are cached for reads on the SSD tier while all writes bypass the cache and go directly to the backing pool.

---

Readonly cache mode in Ceph caches read-accessed objects on the fast tier while routing all writes directly to the backing pool. Unlike writeback mode, there are no dirty objects to flush and no risk of data loss if the cache pool fails. This makes readonly mode safer and simpler, though it only accelerates read-heavy workloads.

## How Readonly Mode Works

```text
Client Write --> Backing Pool (HDD) directly (no cache involvement)

Client Read (cache miss):
  - Object fetched from backing pool
  - Object copied to cache pool (promoted)
  - Future reads served from cache

Client Read (cache hit):
  - Object served from cache pool (SSD, fast)
```

Because writes bypass the cache, the backing pool always has the authoritative copy of every object. Cache objects are always clean (never dirty), so cache loss means reduced performance but never data loss.

## Prerequisites and Pool Setup

```bash
# Backing HDD pool
ceph osd pool create backing-pool 128 128 replicated
ceph osd pool set backing-pool crush_rule hdd-rule

# Cache SSD pool
ceph osd pool create cache-ssd 64 64 replicated
ceph osd pool set cache-ssd crush_rule ssd-rule
```

## Configuring Readonly Cache

```bash
# Add cache tier relationship
ceph osd tier add backing-pool cache-ssd

# Set readonly cache mode
ceph osd tier cache-mode cache-ssd readonly

# Set overlay so clients use cache pool by default
ceph osd tier set-overlay backing-pool cache-ssd
```

## Configuring Promotion Behavior

In readonly mode, objects must meet a minimum hit threshold before being promoted to cache:

```bash
# Track accesses with bloom filter hit sets
ceph osd pool set cache-ssd hit_set_type bloom
ceph osd pool set cache-ssd hit_set_count 4
ceph osd pool set cache-ssd hit_set_period 3600  # 1 hour windows

# Require 2 hits in different windows before promotion
ceph osd pool set cache-ssd min_read_recency_for_promote 2

# Set cache capacity target
ceph osd pool set cache-ssd target_max_bytes 53687091200  # 50 GiB
ceph osd pool set cache-ssd cache_target_full_ratio 0.8
```

## Verifying Cache Mode

```bash
ceph osd dump | grep "cache_mode"
```

```text
cache_mode: readonly
```

## Monitoring Read Cache Effectiveness

```bash
# Check promotion rate and cache hit ratio
ceph osd pool stats cache-ssd
```

```text
client io 2000 MiB/s rd, 0 MiB/s wr to cache
cache tier io 200 MiB/s promote, 50 MiB/s evict
```

If the promotion rate is 0 and read throughput is low, most reads are hitting the backing pool (cache is not warm or `min_read_recency_for_promote` is too high).

## Use Cases for Readonly Mode

Readonly cache mode is best for:
- Read-heavy object storage (media serving, content delivery)
- Database read replicas backed by Ceph RBD
- Static content served via RGW where writes are infrequent
- Any workload where write performance is not a concern but read latency is

It is NOT suitable for:
- Write-heavy workloads (writes bypass cache entirely)
- Workloads needing both read and write acceleration
- Low-latency random write applications

## Removing a Readonly Cache Tier

Because there are no dirty objects, readonly tier removal is simple:

```bash
# Remove overlay
ceph osd tier remove-overlay backing-pool

# Remove the tier relationship
ceph osd tier remove backing-pool cache-ssd

# Delete cache pool
ceph osd pool delete cache-ssd cache-ssd --yes-i-really-really-mean-it
```

## Summary

Readonly cache mode promotes hot objects to the SSD tier for accelerated reads while routing all writes directly to the backing HDD pool. The absence of dirty objects makes it inherently safer than writeback mode. Configure hit set parameters to control promotion aggressiveness and prevent scan-induced cache pollution. It is ideal for read-heavy workloads where data durability cannot be compromised by cache tier failures.
