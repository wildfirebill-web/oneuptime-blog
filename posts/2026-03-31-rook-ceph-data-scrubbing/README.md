# How to Understand Ceph Data Scrubbing (Light and Deep)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Scrubbing, DataIntegrity, Storage, Kubernetes

Description: Learn how Ceph's light and deep scrubbing processes detect and repair data corruption, bit rot, and replication inconsistencies across OSDs.

---

## What Is Data Scrubbing

Scrubbing is Ceph's background integrity verification process. It detects silent data corruption (bit rot), inconsistencies between replicas, and missing or extra objects. Ceph performs two levels of scrubbing:

- **Light scrubbing**: Compares object metadata (size, attributes) across replicas without reading the full data payload
- **Deep scrubbing**: Reads and checksums the actual data on every replica to detect bit rot

Both types operate at the placement group (PG) level and run automatically in the background during low-activity windows.

## Light Scrubbing

Light scrubs run daily by default. The primary OSD compares object names, sizes, and attributes across all replicas in the acting set. This is fast - typically completing within seconds per PG.

```bash
kubectl exec -it rook-ceph-tools -n rook-ceph -- bash

# Trigger a light scrub on a specific PG
ceph pg scrub 2.1e

# View last scrub time per PG
ceph pg dump | awk '{print $1, $19, $20}' | head -20
```

## Deep Scrubbing

Deep scrubs run weekly by default. The primary OSD reads the full data payload on all replicas and verifies checksums. This is I/O intensive but essential for detecting storage media errors that metadata checks would miss.

```bash
# Trigger a deep scrub on a specific PG
ceph pg deep-scrub 2.1e

# View last deep scrub time
ceph pg dump | awk '{print $1, $21}' | head -20
```

## Scrub Scheduling Configuration

Scrubbing is controlled by several Ceph configuration parameters:

```bash
# View current scrub settings
ceph config get osd osd_scrub_begin_hour
ceph config get osd osd_scrub_end_hour
ceph config get osd osd_deep_scrub_interval
```

Configure scrub windows to avoid peak I/O hours:

```bash
# Only scrub between 11 PM and 6 AM
ceph config set osd osd_scrub_begin_hour 23
ceph config set osd osd_scrub_end_hour 6

# Deep scrub interval in seconds (default 604800 = 7 days)
ceph config set osd osd_deep_scrub_interval 1209600
```

## Monitoring Scrub Status

```bash
# Check for PGs with scrub errors
ceph health detail | grep scrub

# List inconsistent PGs
ceph pg ls inconsistent

# Get detailed scrub error info
ceph pg 2.1e query | python3 -m json.tool | grep -A 10 scrub
```

## Repairing Scrub Errors

When a scrub finds an inconsistency, the PG enters the `inconsistent` state. You can attempt automatic repair:

```bash
# Repair a specific PG
ceph pg repair 2.1e

# Repair all inconsistent PGs
ceph health detail | grep inconsistent | awk '{print $2}' | xargs -I{} ceph pg repair {}
```

If repair fails (e.g., all replicas are corrupted), you may need to restore from backup or delete the corrupted object.

## Disabling Scrubbing Temporarily

During maintenance or high I/O periods, you may want to pause scrubbing:

```bash
# Disable scrubbing on all OSDs
ceph osd set noscrub
ceph osd set nodeep-scrub

# Re-enable scrubbing
ceph osd unset noscrub
ceph osd unset nodeep-scrub
```

## Summary

Ceph's two-tier scrubbing system - light scrubs for metadata consistency and deep scrubs for data integrity - provides continuous background verification of your stored data. Light scrubs catch replication inconsistencies cheaply and frequently, while deep scrubs guard against silent storage media corruption. Managing scrub schedules and monitoring scrub health are essential operational tasks for running reliable Rook-Ceph clusters, especially in large deployments where bit rot risk increases with total storage capacity.
