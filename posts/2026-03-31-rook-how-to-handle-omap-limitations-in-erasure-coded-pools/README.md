# How to Handle Omap Limitations in Erasure Coded Pools

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Erasure Coding, OMAP, Object Storage

Description: Understand why erasure coded pools lack omap support in Ceph and how to architect around this limitation using companion replicated pools or RGW index pools.

---

## What Is Omap

Omap is a per-object key-value store embedded within Ceph's RADOS layer. It is used by higher-level services to store metadata alongside objects:

- **CephFS**: Uses omap to store directory entries and extended attributes
- **RGW**: Uses omap for bucket indexes (listing objects in a bucket)
- **RBD**: Uses omap for parent snapshot references and object maps

Omap entries are stored as key-value pairs associated with a specific RADOS object. They enable efficient listing and lookup without requiring separate metadata objects.

## Why EC Pools Do Not Support Omap

Erasure coding in Ceph works by encoding objects into K+M chunks distributed across different OSDs. The omap key-value store is maintained on the primary OSD and requires coordination across the full stripe for updates.

The core issue is that EC pools are optimized for large object data and do not support the in-place modification semantics that omap requires. Specifically:

```text
EC Pool Limitations:
- No omap read/write support
- No atomic compare-and-swap operations
- No RADOS class method support (some)
- Partial object updates require full RMW cycle
```

## Impact on RGW Bucket Indexes

The most common place this limitation is encountered is RGW. Bucket indexes are stored as omap entries on bucket shard objects. If you place the data pool on EC, you must keep the index pool on a replicated pool:

```bash
# RGW zone configuration typically splits index and data
# Check your zone
radosgw-admin zone get | grep -A 20 placement_pools
```

A typical RGW zone with EC has:

```json
{
  "placement_pools": [
    {
      "key": "default-placement",
      "val": {
        "index_pool": ".rgw.buckets.index",
        "data_pool": ".rgw.buckets",
        "data_extra_pool": ".rgw.buckets.non-ec"
      }
    }
  ]
}
```

The `index_pool` must always be replicated. Only `data_pool` can be EC.

## Creating a Proper RGW EC Setup

```bash
# Create replicated index pool
ceph osd pool create .rgw.buckets.index 64 64 replicated
ceph osd pool application enable .rgw.buckets.index rgw

# Create EC data pool
ceph osd erasure-code-profile set rgw-ec k=4 m=2 plugin=jerasure technique=reed_sol_van
ceph osd pool create .rgw.buckets 256 256 erasure rgw-ec
ceph osd pool application enable .rgw.buckets rgw

# Non-EC pool for objects that cannot use EC (multipart manifest, etc.)
ceph osd pool create .rgw.buckets.non-ec 32 32 replicated
ceph osd pool application enable .rgw.buckets.non-ec rgw
```

## Impact on CephFS

For CephFS, the metadata pool must be replicated because the MDS uses omap to store directory entries:

```bash
# Always replicated - cannot be EC
ceph osd pool create cephfs-metadata 64 64 replicated

# Data pool can be EC (with allow_ec_overwrites)
ceph osd pool create cephfs-data 256 256 erasure my-ec-profile
ceph osd pool set cephfs-data allow_ec_overwrites true
```

## Detecting Omap Usage on EC Pools

If an application tries to use omap on an EC pool, it will receive an error:

```bash
# Test omap on an EC pool (will fail)
rados -p my-ec-pool setomapval testobj key1 value1
# Error: EOPNOTSUPP - Operation not supported
```

Monitor for these errors in logs:

```bash
ceph log last 100 | grep -i omap
```

## Workaround Using Companion Pools

For custom applications using RADOS directly, use a companion replicated pool for omap data:

```bash
# EC pool for large objects
ceph osd pool create app-data 256 256 erasure my-ec-profile

# Replicated pool for metadata/omap
ceph osd pool create app-meta 64 64 replicated

# In your application:
# - Store large data objects in app-data
# - Store metadata/omap in app-meta
```

## Summary

Erasure coded pools in Ceph do not support omap, the per-object key-value store used by CephFS metadata, RGW bucket indexes, and RBD object maps. The standard workaround is to use a split-pool architecture: replicated pools for anything requiring omap (metadata, indexes), and EC pools for actual large object data. This split is already the standard pattern in RGW zone configurations and CephFS filesystem layouts.
