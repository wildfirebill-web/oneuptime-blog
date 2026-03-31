# How to Understand the inconsistent PG State in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Placement Group, Inconsistent, Data Integrity, Storage

Description: Understand the Ceph inconsistent PG state, what causes object inconsistencies, how serious they are, and how to repair them safely.

---

The `inconsistent` PG state means a scrub found differences between object replicas that cannot be automatically resolved. It indicates a potential data integrity issue that requires investigation and repair.

## What inconsistent Means

After a scrub, if the primary OSD finds that a replica copy differs from the authoritative copy (different size, checksum mismatch, or missing), it marks the PG as `inconsistent`. This is a `HEALTH_ERR` condition requiring attention.

Types of inconsistencies:

- **Size mismatch**: an object is a different size on one replica
- **Checksum mismatch**: object data differs between replicas
- **Missing object**: an object present on some replicas is missing on others
- **Shard error**: erasure-coded shard is corrupt

## Checking for Inconsistent PGs

```bash
ceph status
# HEALTH_ERR: X pgs inconsistent

ceph health detail
# pg X.Y is inconsistent

ceph pg dump | grep inconsistent
```

Get details about which objects are affected:

```bash
ceph pg <pg-id> query | jq '.info.stats.stat_sum.num_objects_inconsistent'
```

## Finding Affected Objects

```bash
# List inconsistent objects in a PG
rados list-inconsistent-pg mypool

# Get details about inconsistent objects
rados list-inconsistent-obj <pg-id> --pool mypool
```

Example output shows which OSDs have inconsistent copies.

## Repairing Inconsistent PGs

Ceph can automatically repair most inconsistencies by copying from the authoritative replica:

```bash
ceph pg repair <pg-id>
```

Monitor the repair:

```bash
watch ceph health detail | grep "inconsistent"
```

If the repair was successful, the `inconsistent` flag clears.

## When Repair Fails

If `ceph pg repair` cannot resolve the inconsistency (all copies are equally valid or equally corrupt), you may need to manually inspect:

```bash
# Identify the authoritative replica
ceph pg <pg-id> query | jq '.peer_info'

# Check the object on each OSD node
rados -p mypool get <object-name> /tmp/copy1.bin  # from OSD 1
# Compare checksums
md5sum /tmp/copy1.bin /tmp/copy2.bin
```

## Causes of Inconsistency

- Bit rot on a disk
- Hardware memory errors during write
- Storage firmware bugs
- Incomplete writes due to power failure (before journal commit)
- Network corruption during replication

Check OSD hardware health:

```bash
smartctl -a /dev/sdb
dmesg | grep -i "error\|bad\|sector"
```

## Preventing Inconsistencies

```bash
# Ensure deep scrubs run regularly
ceph config get osd osd_deep_scrub_interval

# Enable checksums on new pools
ceph osd pool create mypool 128 128 replicated
ceph osd pool set mypool osd_pool_default_flag_hashpspool true
```

## Summary

Inconsistent PGs indicate object replica mismatches discovered during scrubbing. They are a `HEALTH_ERR` condition that requires prompt attention. Most inconsistencies can be repaired automatically with `ceph pg repair`, which overwrites the corrupt replica with the authoritative one. If automatic repair fails, manual investigation of the affected objects and their OSD hardware is necessary.
