# How to Set Up Readproxy Cache Mode in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, CacheTiering, Storage, Performance

Description: Learn how to configure readproxy cache mode in Ceph, where the cache tier proxies read requests to the backing pool without caching data locally, and writes bypass the cache entirely.

---

Readproxy is the simplest and safest cache tier mode in Ceph. In this mode, the cache pool acts as a proxy layer - read requests pass through to the backing pool, and no data is actually stored in the cache pool. Writes go directly to the backing pool. This mode is useful as an intermediate step when removing a writeback cache tier.

## How Readproxy Works

```text
Client Read Request
  |
  v
Cache Pool (readproxy mode)
  |  - No promotion
  |  - No local caching
  |  - Proxies request to backing pool
  v
Backing Pool (HDD)
  |
  v
Data returned to client through cache pool
```

In readproxy mode:
- The cache pool routes read requests to the backing pool
- No objects are stored in or promoted to the cache pool
- Write operations go directly to the backing pool
- Dirty objects from a previous writeback phase are still flushed if present

## Primary Use Case: Writeback Tier Removal

Readproxy is typically used as a transition mode when removing a writeback cache tier:

1. Change from `writeback` to `readproxy` - new writes go to backing pool
2. Wait for all dirty objects in cache to be flushed
3. Change from `readproxy` to `none` (or remove tier entirely)

This prevents data loss during cache removal.

## Setting Up Readproxy Mode

First, set up pools and the tier relationship:

```bash
# Backing pool
ceph osd pool create backing-pool 128 128 replicated

# Cache pool (small, no data stored)
ceph osd pool create cache-pool 32 32 replicated

# Add cache tier
ceph osd tier add backing-pool cache-pool

# Set readproxy mode
ceph osd tier cache-mode cache-pool readproxy

# Set overlay
ceph osd tier set-overlay backing-pool cache-pool
```

## Verifying Readproxy Mode

```bash
ceph osd dump | grep -A 3 "pool 'cache-pool'"
```

```text
pool 2 'cache-pool' replicated ...
        cache_mode: readproxy
        tier_of 1
```

## Transitioning from Writeback to Readproxy

```bash
# Change an existing writeback cache to readproxy
ceph osd tier cache-mode cache-pool readproxy
```

Immediately after this change, check for remaining dirty objects:

```bash
rados -p cache-pool ls | head -20
# Should show decreasing objects as they are flushed
```

Monitor flush progress:

```bash
watch "ceph df | grep cache-pool"
```

## Monitoring During Readproxy Phase

Since no new objects are being written to the cache pool, the primary metric to watch is the flush of remaining dirty objects:

```bash
ceph osd pool stats cache-pool
```

```text
cache tier io 500 MiB/s flush, 0 MiB/s promote
```

Promotion rate drops to 0 in readproxy mode. Flush rate continues until all dirty objects are gone.

## When to Use Readproxy in Production

Beyond the removal use case, readproxy can be useful when:

- You want the cache pool to act as a network proximity layer (e.g., cache pool on a different rack) without the complexity of writeback
- You are transitioning a cluster to remove cache tiering in favor of application-layer caching
- You need to disable write caching temporarily for maintenance

## Summary

Readproxy cache mode proxies reads through the cache pool to the backing pool without storing any data locally, while writes go directly to the backing pool. It is primarily used as a safe intermediate step when removing a writeback cache tier. Switching to readproxy stops new dirty objects from accumulating and allows existing dirty objects to be flushed before completing the tier removal.
