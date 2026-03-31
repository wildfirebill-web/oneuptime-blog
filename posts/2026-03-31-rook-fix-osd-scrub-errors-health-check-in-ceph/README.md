# How to Fix OSD_SCRUB_ERRORS Health Check in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, OSD, Health Check, Scrub

Description: Learn how to diagnose and fix OSD_SCRUB_ERRORS in Ceph, which indicates scrub operations detected data inconsistencies on one or more OSDs.

---

## What Is OSD_SCRUB_ERRORS?

`OSD_SCRUB_ERRORS` is a Ceph health warning that appears when a scrub or deep scrub operation finds inconsistencies in data stored on an OSD. Scrubbing is Ceph's data integrity verification process - it compares object copies across replicas to detect corruption, missing objects, or checksum mismatches.

Unlike `PG_DAMAGED` (which flags the PG), `OSD_SCRUB_ERRORS` flags the specific OSDs that reported errors during scrubbing.

## Identifying the Affected OSDs

```bash
ceph health detail
```

Example output:

```text
[WRN] OSD_SCRUB_ERRORS: 3 OSD(s) have scrub errors
    osd.1 had scrub errors
    osd.5 had scrub errors
    osd.9 had scrub errors
```

Check PG-level inconsistencies:

```bash
ceph pg dump_stuck inconsistent
```

Get OSD-specific error counts:

```bash
ceph osd scrub-error-check osd.1
```

## Steps to Investigate and Fix

### Step 1 - Run a Deep Scrub on Affected PGs

A deep scrub reads object data (not just metadata) and recomputes checksums:

```bash
ceph osd deep-scrub osd.1
```

Or scrub specific PGs:

```bash
ceph pg deep-scrub 1.0
ceph pg deep-scrub 1.4
```

Monitor scrub progress:

```bash
ceph pg dump | grep scrub
```

### Step 2 - Attempt PG Repair

If inconsistencies are found, attempt automatic repair:

```bash
ceph pg repair 1.0
```

This instructs Ceph to use the authoritative copy to overwrite the inconsistent copy.

### Step 3 - Check OSD Hardware

Persistent scrub errors often indicate disk hardware problems. Check SMART data:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- smartctl -H /dev/sdb
```

Check kernel logs on the OSD node:

```bash
journalctl -u ceph-osd@1 | grep -i error
dmesg | grep -i "I/O error\|sector"
```

### Step 4 - Check Memory for Bit Flips

Memory errors (RAM corruption) can cause scrub errors. Run a memory test on the OSD node:

```bash
memtester 1G 1
```

Or check ECC memory status:

```bash
edac-util -s 4
```

### Step 5 - Replace the Problematic OSD

If hardware is confirmed bad:

```bash
ceph osd out osd.1
# wait for rebalance
ceph osd down osd.1
ceph osd rm osd.1
ceph osd crush remove osd.1
ceph auth del osd.1
```

## Tuning Scrub Behavior

Adjust scrub scheduling to avoid peak hours:

```bash
ceph config set osd osd_scrub_begin_hour 22
ceph config set osd osd_scrub_end_hour 6
ceph config set osd osd_scrub_min_interval 86400
ceph config set osd osd_deep_scrub_interval 604800
```

## Summary

`OSD_SCRUB_ERRORS` indicates data inconsistencies found during scrub operations. Fix it by running deep scrubs on affected PGs, using `ceph pg repair` for automatic remediation, and investigating hardware issues on OSDs that repeatedly report errors. Persistent errors on the same OSD usually point to failing disk hardware that needs replacement.
