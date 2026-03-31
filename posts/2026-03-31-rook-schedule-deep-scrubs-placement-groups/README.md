# How to Schedule Deep Scrubs for Placement Groups in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Placement Group, Scrubbing, Data Integrity

Description: Learn how to schedule and manage deep scrubs for Ceph Placement Groups in Rook to detect bit rot and ensure long-term data integrity.

---

## Deep Scrub vs Regular Scrub

Regular scrubs compare only object metadata (size, attributes) between replicas. Deep scrubs read and compare actual object data, computing and verifying checksums end-to-end. Deep scrubs detect:

- Silent bit rot on disk
- Media errors that did not trigger read errors
- Checksum mismatches between replicas
- Objects corrupted at the filesystem level

Deep scrubs are more I/O intensive and slower, but essential for long-term data integrity in production clusters.

## Default Deep Scrub Frequency

By default, Ceph performs a deep scrub on each PG every 7 days. The schedule is controlled by:

- `osd_deep_scrub_interval`: Minimum time between deep scrubs (default: 7 days in seconds = 604800)
- `osd_scrub_max_interval`: Maximum time before a scrub is forced (default: 7 days)

## Initiating a Manual Deep Scrub

Deep scrub a specific PG:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg deep-scrub 1.2a
```

Deep scrub all PGs in a pool:

```bash
for pg in $(kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg ls-by-pool mypool 2>/dev/null | awk 'NR>1 {print $1}'); do
  kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
    ceph pg deep-scrub $pg
  sleep 2
done
```

## Scheduling Deep Scrubs to Off-Peak Hours

Configure deep scrubs to only run during maintenance windows:

```bash
# Allow deep scrubs only from 1am to 5am
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_scrub_begin_hour 1

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_scrub_end_hour 5
```

Note: Both scrub and deep scrub respect these time windows.

## Adjusting Deep Scrub Frequency

Increase frequency for high-value or compliance-sensitive data:

```bash
# Deep scrub every 3 days (259200 seconds)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_deep_scrub_interval 259200
```

For archival pools where data changes rarely:

```bash
# Deep scrub every 14 days
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_deep_scrub_interval 1209600
```

## Checking Last Deep Scrub Timestamps

Find PGs that haven't been deep scrubbed recently:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg dump 2>/dev/null | \
  awk 'NF>20 {print $1, "deep_scrub:", $19}' | \
  sort -k3 | head -20
```

## Limiting Deep Scrub I/O Impact

Control how fast deep scrubs read data to limit interference with clients:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_scrub_sleep 0.1

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_scrub_chunk_min 1

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_scrub_chunk_max 5
```

## Disabling Deep Scrubs Temporarily

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd set nodeep-scrub
```

Re-enable when ready:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd unset nodeep-scrub
```

## Summary

Deep scrubs provide end-to-end data integrity verification by reading and checksumming all object data. Schedule them during off-peak hours using `osd_scrub_begin_hour` and `osd_scrub_end_hour`. Adjust frequency via `osd_deep_scrub_interval` based on your data change rate and integrity requirements. Use `osd_scrub_sleep` to throttle I/O impact during business hours.
