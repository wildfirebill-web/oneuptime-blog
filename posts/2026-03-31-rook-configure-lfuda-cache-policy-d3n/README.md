# How to Configure LFUDA Cache Policy for D3N

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, D3N, LFUDA, Cache, Eviction, RGW

Description: Configure the LFUDA (Least Frequently Used with Dynamic Aging) cache eviction policy for D3N in Ceph RGW to maximize cache efficiency for mixed workloads.

---

## Overview

D3N uses LFUDA (Least Frequently Used with Dynamic Aging) as its cache eviction policy. LFUDA improves on basic LFU by applying a dynamic aging factor that prevents old frequently-accessed objects from monopolizing the cache indefinitely. Understanding and tuning LFUDA ensures your D3N cache stays filled with the objects most likely to be requested next.

## How LFUDA Works

LFUDA assigns each cached object a score based on:

- **Access frequency** - how many times the object has been requested
- **Dynamic aging** - a time-based penalty that reduces scores of objects that haven't been accessed recently

When the cache is full and a new object needs to be stored, the object with the lowest score is evicted. This ensures recently popular objects remain while objects that were popular long ago get aged out.

## Default LFUDA Behavior

LFUDA is the default eviction policy for D3N and requires no explicit configuration to enable. The key tunable parameter is the aging factor:

```bash
# Check current LFUDA aging configuration
ceph config get client.rgw.myzone rgw_d3n_l1_eviction_policy
```

## Configuring LFUDA Parameters

```bash
# Set eviction policy explicitly (lfuda is default)
ceph config set client.rgw.myzone rgw_d3n_l1_eviction_policy lfuda

# Set the cache inflation factor (higher = more weight on frequency vs recency)
ceph config set client.rgw.myzone d3n_l1_datacache_size 21474836480
```

In `ceph.conf`:

```ini
[client.rgw.myzone]
d3n_l1_local_datacache_enabled = true
d3n_l1_datacache_persistent_path = /var/lib/ceph/rgw/cache
d3n_l1_datacache_size = 21474836480
rgw_d3n_l1_eviction_policy = lfuda
```

## Monitoring LFUDA Eviction Activity

```bash
# View cache eviction counters
ceph daemon rgw.myzone perf dump | python3 -m json.tool | grep -i "evict\|cache"

# Check how full the cache is
du -sh /var/lib/ceph/rgw/cache
df -h /var/lib/ceph/rgw/cache
```

## Tuning Cache Size to Reduce Evictions

The most effective way to improve LFUDA efficiency is to size the cache appropriately. If you see frequent evictions:

```bash
# Check eviction rate
journalctl -u ceph-radosgw@rgw.myzone --no-pager | grep -i evict | wc -l

# Increase cache size if the device has space
ceph config set client.rgw.myzone d3n_l1_datacache_size 42949672960
```

## Comparing with LRU

While LRU (Least Recently Used) evicts the oldest object regardless of access frequency, LFUDA keeps objects that are accessed repeatedly even if there was a gap in access. For media streaming, ML dataset reads, or backup restoration workflows, LFUDA outperforms LRU significantly:

| Workload | LRU Efficiency | LFUDA Efficiency |
|---|---|---|
| Streaming media | Medium | High |
| Random one-time reads | High | Medium |
| Repeated dataset access | Low | High |
| Mixed workloads | Medium | High |

## Summary

LFUDA is D3N's default and recommended eviction policy, combining access frequency with dynamic aging to keep the most relevant objects in cache. The most important tuning lever is cache size - a larger cache reduces evictions and improves hit rates for all workloads. Monitor eviction counters via the admin socket and RGW logs to determine if your cache is undersized.
