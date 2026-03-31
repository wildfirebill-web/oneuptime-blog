# How to Scrub Placement Groups in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Placement Group, Scrubbing, Data Integrity

Description: Learn how to initiate and manage Placement Group scrubbing in Ceph to detect data corruption and inconsistencies in Rook clusters.

---

## What Is PG Scrubbing?

Ceph scrubbing is an integrity check process where the primary OSD compares object metadata and checksums across all replicas of a Placement Group. There are two types:

- **Scrub (light scrub)**: Compares object metadata (size, attributes) across replicas. Fast, runs frequently.
- **Deep scrub**: Reads and compares actual object data across replicas. Slower but detects bit rot and checksum mismatches.

Scrubbing is the primary mechanism for detecting silent data corruption in a Ceph cluster.

## Checking Scrub Status

View last scrub times across all PGs:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg dump 2>/dev/null | \
  awk 'NF>20 {print $1, "last_scrub:", $18}' | head -20
```

Check for scrub errors:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph health detail | grep -i scrub
```

## Initiating a Manual Scrub

Scrub a specific PG:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg scrub 1.2a
```

Scrub all PGs in a pool:

```bash
for pg in $(kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg ls-by-pool mypool 2>/dev/null | awk 'NR>1 {print $1}'); do
  kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph pg scrub $pg
done
```

## Configuring Scrub Schedule

Control when automatic scrubs run by setting time windows:

```bash
# Only scrub between 2am and 6am
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_scrub_begin_hour 2

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_scrub_end_hour 6
```

## Controlling Scrub Load Impact

Limit how many scrubs run simultaneously:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_max_scrubs 1

# Reduce scrub priority to minimize client impact
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_scrub_priority 5
```

## Disabling Scrubbing Temporarily

Disable scrubbing during critical operations:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd set noscrub

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd set nodeep-scrub
```

Re-enable after the operation completes:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd unset noscrub

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd unset nodeep-scrub
```

## Responding to Scrub Errors

If scrubbing detects inconsistencies:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph health detail | grep -i inconsistent
```

Repair a PG with inconsistencies:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg repair <pgid>
```

## Summary

PG scrubbing detects data corruption by comparing replicas across OSDs. Initiate manual scrubs with `ceph pg scrub <pgid>` or schedule automatic scrubs using `osd_scrub_begin_hour` and `osd_scrub_end_hour`. Limit scrub concurrency with `osd_max_scrubs` to protect production I/O. Use `ceph pg repair` when scrubs detect inconsistencies between replicas.
