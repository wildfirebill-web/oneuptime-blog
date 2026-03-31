# How to Migrate Ceph Data Between Different Pool Types

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Migration, Pool, Storage

Description: Learn how to move Ceph data between replicated and erasure-coded pools, or between pools with different CRUSH rules, using rados cppool and RBD export/import.

---

Storage requirements evolve over time. You might need to move data from a replicated pool to an erasure-coded pool to save space, or migrate to a pool on different hardware using different CRUSH rules. Ceph provides several tools for cross-pool data migration.

## When to Migrate Between Pool Types

Common reasons to migrate pools:
- Switch from 3-way replication to erasure coding to reduce storage overhead
- Move hot data to SSD-backed pools and cold data to HDD pools
- Separate tenant data onto dedicated pools with quotas
- Change CRUSH rules to use different failure domains

## Option 1 - RADOS Copy Pool (Same Object Type)

For pools containing RADOS objects (not RBD or CephFS), use cppool:

```bash
rados cppool source-pool dest-pool
```

This copies all objects. Monitor progress:

```bash
watch "rados df | grep -E 'source-pool|dest-pool'"
```

Note: cppool does not work for pools with snapshots or linked objects.

## Option 2 - RBD Export and Import

For RBD block volumes:

```bash
# Export from source pool
rbd export source-pool/myvolume /tmp/myvolume.img

# Import to destination pool
rbd import /tmp/myvolume.img dest-pool/myvolume
```

For large volumes, use incremental export:

```bash
rbd snap create source-pool/myvolume@snap1
rbd export-diff source-pool/myvolume@snap1 /tmp/vol-snap1.diff

# Import diff to destination
rbd snap create dest-pool/myvolume@snap1
rbd import-diff /tmp/vol-snap1.diff dest-pool/myvolume
```

## Option 3 - Using Object Cache Tier for Pool Migration

For live migration without downtime, use cache tiering temporarily:

```bash
# Source pool becomes the backing tier
# Destination pool becomes the cache tier (writeback mode)

ceph osd tier add dest-pool source-pool
ceph osd tier cache-mode dest-pool writeback
ceph osd tier set-overlay source-pool dest-pool

# Data moves naturally as it is accessed
# Watch migration progress
rados -p source-pool df
```

When migration is complete:

```bash
ceph osd tier cache-mode dest-pool proxy
ceph osd tier remove-overlay source-pool
ceph osd tier remove dest-pool source-pool
```

## Option 4 - CephFS Data Migration

For CephFS data on a different pool, create a new layout:

```bash
# Create new pool
ceph osd pool create new-data-pool 64

# Add pool to filesystem
ceph fs add_data_pool myfs new-data-pool

# Set new layout on a directory
setfattr -n ceph.dir.layout.pool -v new-data-pool /mnt/cephfs/mydir

# New files in mydir go to new-data-pool
# Existing files stay in old pool until rewritten
```

## Verifying Migration

```bash
# Check object counts
rados df | grep -E "source-pool|dest-pool"

# For RBD
rbd info dest-pool/myvolume
rbd du dest-pool
```

## Summary

Migrating data between Ceph pool types can be accomplished through rados cppool for RADOS objects, RBD export/import for block volumes, and CephFS layout changes for filesystem data. For zero-downtime migration of active data, the cache tiering approach provides a live migration path without application interruption.
