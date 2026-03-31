# How to Fix Inconsistent Objects Found by Scrubbing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Scrubbing, Repair, Data Integrity, Kubernetes, Recovery

Description: Learn how to repair inconsistent objects detected by Ceph scrubbing, using pg repair, manual object replacement, and recovery procedures for different types of inconsistencies.

---

## Understanding Object Inconsistencies

When Ceph scrubbing finds inconsistent objects, it means that the copies of an object stored on different OSDs do not agree. This can indicate:

- **Bit rot**: Silent data corruption on a disk
- **Software bug**: A Ceph bug that wrote incorrect data
- **Hardware failure**: A failing disk writing incorrect data
- **Race condition**: A very rare case of interrupted writes

Before repairing, identify the root cause to prevent recurrence.

## Step 1 - Identify the Inconsistent PGs

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph health detail | grep "inconsistent"

# List all inconsistent PGs
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg dump | awk '$1 ~ /[0-9]/ && /inconsistent/ {print $1}'
```

## Step 2 - Gather Information Before Repairing

```bash
# Get detailed information about the inconsistent PG
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg 3.1a query | python3 -m json.tool > /tmp/pg-3-1a.json

# Check which objects are inconsistent
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rados list-inconsistent-pg my-pool

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rados list-inconsistent-obj 3.1a
```

## Step 3 - Use pg repair for Automated Resolution

Ceph can automatically repair most inconsistencies by overwriting the inconsistent replica with the authoritative copy:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg repair 3.1a
```

Monitor repair progress:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  watch ceph health detail
```

After repair completes, verify the inconsistency is resolved:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg deep-scrub 3.1a

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph health detail | grep "3.1a"
```

## Step 4 - Manual Repair for Unresolvable Inconsistencies

If `pg repair` does not fix the issue, you may need to manually intervene:

```bash
# List the specific inconsistent objects
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rados list-inconsistent-obj 3.1a | python3 -m json.tool

# Export the object from the authoritative OSD
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rados -p my-pool get <object-name> /tmp/object-backup

# Re-put the object to force all replicas to update
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rados -p my-pool put <object-name> /tmp/object-backup
```

## Step 5 - Handling Missing Objects

If an object is missing from a replica (not just inconsistent data):

```bash
# Check if the object exists at all
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rados -p my-pool stat <object-name>

# If the object exists, force re-replication
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg repair 3.1a
```

## Investigating Root Cause

After repair, investigate why the inconsistency occurred:

```bash
# Check for hardware errors on the OSDs involved
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd df tree | grep -E "osd\.[0-9]+"

# Check OSD logs for I/O errors
kubectl -n rook-ceph logs rook-ceph-osd-3-xxxxx | grep -E "ERROR|corrupt|checksum"
```

## Summary

Fixing inconsistent objects detected by Ceph scrubbing usually starts with `ceph pg repair`, which overwrites the inconsistent replica with the authoritative copy. For objects that cannot be automatically repaired, use `rados get/put` to manually force re-replication. Always investigate the root cause after repair - recurring inconsistencies on the same OSD indicate failing hardware that should be replaced before it causes further data integrity issues.
