# How to Fix 'scrub errors' Detected During Data Scrubbing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Troubleshooting, Scrub, Data Integrity, OSD, Recovery

Description: Diagnose and repair Ceph scrub errors by identifying inconsistent objects, running repairs, and replacing failing hardware.

---

## Introduction

Ceph scrubbing compares object replicas across OSDs to detect bit rot, silent corruption, or inconsistency. When scrub errors are found, the cluster reports `HEALTH_ERR` with inconsistent PGs. These errors require prompt attention to prevent data loss.

## Identifying Scrub Errors

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

Example:

```
HEALTH_ERR 2 scrub errors; 1 pgs inconsistent
pg 1.a is active+clean+inconsistent
```

List inconsistent PGs:

```bash
ceph pg dump | grep inconsistent
```

## Step 1 - Get Scrub Error Details

```bash
ceph pg 1.a query | python3 -m json.tool | grep -A5 "errors"
```

Identify which objects are inconsistent:

```bash
rados list-inconsistent-pg replicapool
rados list-inconsistent-obj 1.a --format=json | python3 -m json.tool
```

## Step 2 - Attempt Automatic Repair

Ceph can repair many scrub errors automatically by using the authoritative copy:

```bash
ceph pg repair 1.a
```

Wait for repair to complete:

```bash
watch "ceph pg 1.a query | grep state"
```

The PG should transition from `inconsistent` back to `active+clean`.

## Step 3 - Verify Repair Success

```bash
ceph health
ceph pg dump | grep -v clean | grep -v "^$"
```

If the PG is still inconsistent after repair, the error may require manual investigation.

## Step 4 - Investigate Hardware

Persistent scrub errors often indicate a failing disk:

```bash
# Check disk health on the OSD node
smartctl -a /dev/sdb
```

Look for reallocated sectors, pending sectors, or CRC errors in the SMART output.

If the disk is failing, the OSD must be replaced:

```bash
# Mark the OSD out to start rebalancing
ceph osd out osd.3

# Wait for rebalancing
watch "ceph -s"

# Then destroy and remove the OSD
ceph osd down osd.3
ceph osd purge osd.3 --yes-i-really-mean-it
```

## Step 5 - Force Deep Scrub on Repaired PGs

After repair, run a deep scrub to verify all replicas are consistent:

```bash
ceph pg deep-scrub 1.a
```

Monitor deep scrub progress:

```bash
ceph pg stat
ceph status | grep scrub
```

## Preventing Future Scrub Errors

Ensure scrubbing runs regularly:

```bash
ceph config set osd osd_scrub_min_interval 86400   # daily minimum
ceph config set osd osd_scrub_max_interval 604800  # weekly maximum
ceph config set osd osd_deep_scrub_interval 604800 # weekly deep scrub
```

## Summary

Ceph scrub errors indicate object inconsistency between replicas, which can result from bit rot, silent disk corruption, or hardware failure. The standard fix is running `ceph pg repair` to let Ceph use the authoritative copy. Persistent errors warrant SMART disk health checks and OSD replacement if the underlying hardware is degraded.
