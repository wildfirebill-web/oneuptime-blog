# How to Fix "deep-scrub errors found" in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Troubleshooting, Deep Scrub, Data Integrity, Repair, OSD

Description: Fix deep-scrub errors in Ceph by running PG repair operations, identifying corrupted objects, and replacing faulty hardware.

---

## Introduction

Ceph deep scrubbing compares object data checksums across all replicas to detect silent corruption. Unlike regular scrubbing (which only checks metadata), deep scrubbing reads and verifies actual data. When deep-scrub errors are found, they indicate data corruption that needs repair.

## Identifying Deep Scrub Errors

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

Example:

```
HEALTH_ERR 1 pgs inconsistent; 2 scrub errors
pg 2.f is active+clean+inconsistent, last_scrub_stamp 2026-03-31
2 errors found in 2 objects in 1 pg
```

Get details on the affected PG:

```bash
ceph pg 2.f query | python3 -m json.tool | grep -A10 "errors"
```

## Step 1 - List Inconsistent Objects

```bash
rados list-inconsistent-pg replicapool
rados list-inconsistent-obj 2.f --format=json-pretty
```

Output identifies which objects have checksum mismatches and on which OSDs.

## Step 2 - Attempt Automatic Repair

```bash
ceph pg repair 2.f
```

Ceph uses the majority of copies to determine the authoritative version and overwrites the corrupted copy.

Monitor repair:

```bash
watch "ceph health"
ceph pg stat
```

## Step 3 - Verify Repair Succeeded

Run another deep scrub after repair:

```bash
ceph pg deep-scrub 2.f

# Wait for deep scrub to complete
watch "ceph pg 2.f query | python3 -m json.tool | grep state"
```

If the PG returns to `active+clean`, repair was successful.

## Step 4 - Handle Unrepairable Objects

If repair fails because all copies are corrupted, the object data is permanently lost:

```bash
rados list-inconsistent-obj 2.f --format=json-pretty | python3 -m json.tool
```

Identify the object name and check if it can be restored from a backup:

```bash
# Remove the corrupted object (only if confirmed unrecoverable)
rados -p replicapool rm <object-name>
```

Then re-ingest from backup.

## Step 5 - Check Disk Health

Persistent deep scrub errors indicate hardware failure:

```bash
# On the OSD node
smartctl -a /dev/sdb
dmesg | grep -i "error\|failure\|bad sector"
```

If the disk has bad sectors, the OSD must be replaced:

```bash
ceph osd out osd.4
kubectl -n rook-ceph delete pod rook-ceph-osd-4-<pod-id>
ceph osd purge osd.4 --yes-i-really-mean-it
```

## Scheduling Regular Deep Scrubs

Deep scrubs run automatically but you can configure the schedule:

```bash
ceph config set osd osd_deep_scrub_interval 604800   # weekly
ceph config set osd osd_scrub_chunk_max 25
ceph config set osd osd_deep_scrub_randomize_ratio 0.15
```

Trigger a manual deep scrub on a pool:

```bash
ceph osd pool scrub replicapool deep
```

## Summary

Deep-scrub errors indicate real data corruption that may result from bit rot, disk failure, or memory errors. The first response is running `ceph pg repair`, which uses the majority of copies to recover. If repair fails, check disk SMART health and replace faulty hardware. Always verify repairs with a follow-up deep scrub before considering the issue resolved.
