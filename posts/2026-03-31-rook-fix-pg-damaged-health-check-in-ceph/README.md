# How to Fix PG_DAMAGED Health Check in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Placement Group, Health Check, Data Integrity

Description: Learn how to fix PG_DAMAGED in Ceph, a critical health error indicating placement groups have inconsistencies or damage found during scrubbing operations.

---

## What Is PG_DAMAGED?

`PG_DAMAGED` is a critical Ceph health error that fires when a scrub or deep scrub finds inconsistent or damaged data in a Placement Group. This can mean object checksums don't match, objects are missing, or data is corrupt across replicas. It is one of the most serious health alerts in Ceph because it indicates actual data integrity problems.

This alert requires immediate attention. Damaged PGs can lead to data loss if the inconsistency is not repaired.

## Checking Health Detail

```bash
ceph health detail
```

Example output:

```text
[ERR] PG_DAMAGED: Inconsistent PGs found
    pg 3.4 is active+clean+inconsistent
    pg 3.4 has 2 inconsistent objects
```

Get full inconsistency details:

```bash
ceph pg 3.4 query | python3 -m json.tool
```

Check the OSD scrub error logs:

```bash
ceph pg dump_stuck inconsistent
```

## Understanding Inconsistency Types

Ceph reports several types of inconsistencies:

- `shard_missing` - an object replica is missing entirely
- `stat_error` - object size or attributes don't match
- `read_error` - the OSD failed to read the object
- `data_digest_mismatch` - checksums don't match between replicas
- `omap_digest_mismatch` - omap (key-value) data checksums don't match

## Fix Steps

### Step 1 - Attempt Automatic Repair

Ceph can attempt to repair inconsistent PGs by choosing the "authoritative" copy:

```bash
ceph pg repair 3.4
```

Monitor repair progress:

```bash
watch ceph pg 3.4 query
```

After repair completes, re-scrub to verify:

```bash
ceph pg scrub 3.4
ceph pg deep-scrub 3.4
```

### Step 2 - Check for Hardware Issues

If repair fails, the underlying disk may have hardware errors. Check SMART data:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- smartctl -a /dev/sdb
```

Check kernel dmesg for disk errors on the affected OSD node:

```bash
dmesg | grep -i "error\|failed\|sector"
```

### Step 3 - Identify the Authoritative OSD

If automatic repair fails, manually identify which OSD has the good copy:

```bash
ceph pg 3.4 query | grep -A 20 "acting"
```

Log into each acting OSD and compare object checksums using `rados`:

```bash
rados -p <pool-name> get <object-name> /tmp/obj_copy
md5sum /tmp/obj_copy
```

### Step 4 - Remove and Re-add Inconsistent OSD

If a specific OSD consistently produces inconsistencies, it may have hardware corruption. Remove it:

```bash
ceph osd out osd.2
# wait for rebalance
ceph osd rm osd.2
ceph osd crush remove osd.2
ceph auth del osd.2
```

## Summary

`PG_DAMAGED` is a critical alert indicating real data inconsistencies found during scrubbing. Start with `ceph pg repair` for automatic repair, then investigate hardware issues with SMART data if repair fails. For persistent inconsistencies, identify and remove the corrupt OSD. Regular deep scrubs (`osd_deep_scrub_interval`) are the best prevention strategy.
