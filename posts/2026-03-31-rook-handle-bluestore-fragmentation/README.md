# How to Handle BlueStore Fragmentation in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, BlueStore, Fragmentation, OSD, Performance, Maintenance, Tuning

Description: Understand BlueStore fragmentation in Ceph OSDs and learn how to diagnose, prevent, and remediate excessive fragmentation.

---

## Overview

BlueStore fragmentation occurs when the block device's free space becomes scattered into many small non-contiguous extents. Fragmentation degrades write performance because BlueStore must find and stitch together multiple small free extents for each new write. This guide explains what causes BlueStore fragmentation and how to manage it.

## How BlueStore Fragmentation Develops

BlueStore uses a free-list allocator to manage space on the block device. Over time, as objects are written, overwritten, and deleted, the free space becomes fragmented:

1. Objects are written in allocation units
2. Some objects are deleted, freeing their extents
3. New objects fill some but not all freed extents
4. Remaining free space is scattered in small chunks

## Checking Fragmentation Level

Ceph provides a health warning for excessive fragmentation:

```bash
# Check for fragmentation health warnings
ceph health detail | grep BLUESTORE_FRAGMENTATION

# Get fragmentation score per OSD (0.0 = no fragmentation, 1.0 = maximum)
ceph osd df | grep -v "^osd\." | head -5
```

Get a detailed fragmentation report:

```bash
# For a specific OSD
ceph daemon osd.0 bluestore allocator score block
```

Example output:

```
{
  "allocator_name": "block",
  "alloc_unit": 4096,
  "capacity": 107374182400,
  "num_free": 52428800,
  "fragmentation_rating": 0.42
}
```

## Thresholds and Health Warnings

Ceph issues health warnings at these fragmentation levels:

| Score | Status |
|---|---|
| 0.0 - 0.7 | HEALTH_OK |
| 0.7 - 0.9 | HEALTH_WARN |
| 0.9 - 1.0 | HEALTH_ERR |

```bash
# View current fragmentation thresholds
ceph config show osd.0 bluestore_fragmentation_threshold
```

## Preventing Fragmentation

### Set Appropriate min_alloc_size

Using a `min_alloc_size` that matches your workload's write size prevents creation of many small scattered extents:

```bash
ceph config set osd bluestore_min_alloc_size_hdd 65536
```

### Enable Deferred Compaction

BlueStore can compact small free extents during idle periods:

```bash
ceph config set osd bluestore_deferred_batch_ops 32
```

## Remediating Fragmentation

### Method 1 - Deep Scrub with Compaction

Force a deep scrub which triggers BlueStore cleanup:

```bash
ceph osd deep-scrub osd.0
```

### Method 2 - Rebalance the OSD

Temporarily mark the OSD out and back in to trigger data redistribution:

```bash
# Warning: this moves data off the OSD
ceph osd out 0
# Wait for data to migrate to other OSDs
# Then bring back in
ceph osd in 0
```

### Method 3 - Online Compaction

For RocksDB metadata fragmentation, trigger a manual compaction:

```bash
ceph daemon osd.0 compact
```

This compacts RocksDB but not the main block device allocator.

### Method 4 - OSD Replacement

For severe fragmentation, replace the OSD:

```bash
# Mark OSD out
ceph osd out 0

# Wait for data migration
watch ceph osd df | grep "osd.0"

# Remove and recreate the OSD
ceph osd purge 0 --yes-i-really-mean-it
# Then provision a new OSD on the same device
```

## Monitoring Fragmentation Over Time

Create a monitoring script:

```bash
#!/bin/bash
for OSD in $(ceph osd ls); do
  SCORE=$(ceph daemon osd.$OSD bluestore allocator score block 2>/dev/null | \
    python3 -c "import sys,json; print(json.load(sys.stdin).get('fragmentation_rating','N/A'))")
  echo "OSD $OSD: fragmentation_rating=$SCORE"
done
```

## Summary

BlueStore fragmentation develops naturally over time as objects are created and deleted. Monitoring fragmentation scores with `ceph daemon osd.X bluestore allocator score block`, setting appropriate `min_alloc_size` to match your workload, and periodically cycling heavily fragmented OSDs out-and-in keeps fragmentation manageable. For active degradation, triggering deep scrubs or RocksDB compaction provides partial relief without data movement.
