# How to Set and Unset the nodeep-scrub Flag in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, OSD, Scrub, Flag, Maintenance

Description: Learn how to disable Ceph deep scrubbing with the nodeep-scrub flag to reduce disk I/O during busy periods without stopping light scrubs.

---

Deep scrubbing reads and verifies every byte of every object in a PG. This is thorough but I/O intensive. The `nodeep-scrub` flag disables deep scrubs while allowing light scrubs to continue, striking a balance between data integrity and performance.

## Deep Scrub vs Light Scrub

| Aspect | Light Scrub | Deep Scrub |
|--------|-------------|------------|
| What it checks | Metadata, object lists | All object data (byte-by-byte) |
| I/O intensity | Low | High |
| Default interval | Daily | Weekly |
| Flag to disable | noscrub | nodeep-scrub |

## Setting nodeep-scrub

```bash
ceph osd set nodeep-scrub
```

Confirm:

```bash
ceph osd dump | grep flags
ceph status
# HEALTH_WARN: nodeep-scrub flag(s) set
```

## Use Case - Protecting Performance on HDDs

Deep scrubs on spinning disks are especially disruptive because sequential read throughput is limited. During business hours or heavy workloads:

```bash
# Disable deep scrub during the day
ceph osd set nodeep-scrub

# Re-enable for the maintenance window
ceph osd unset nodeep-scrub
```

## Use Case - During Recovery Operations

When a cluster is already performing recovery after a failure, adding deep scrub I/O on top worsens recovery time. Pause deep scrubs until recovery completes:

```bash
ceph osd set nodeep-scrub
# Monitor recovery progress
watch ceph status
# When active+clean:
ceph osd unset nodeep-scrub
```

## Scheduling Deep Scrubs with Time Windows

Rather than manually toggling the flag, configure a time window:

```bash
ceph config set osd osd_scrub_begin_hour 22
ceph config set osd osd_scrub_end_hour 6
ceph config set osd osd_deep_scrub_interval 604800   # 7 days
```

This limits deep scrubs to nighttime hours automatically.

## Checking Deep Scrub Status

View the last deep scrub time for PGs:

```bash
ceph pg dump --format json | jq '.pg_stats[] | {pgid, last_deep_scrub, last_deep_scrub_stamp}'
```

Identify PGs that have not been deep scrubbed recently:

```bash
ceph pg dump | awk 'NR>1 {print $1, $22}' | sort -k2
```

## Manually Triggering a Deep Scrub

After removing the flag, force a deep scrub on a specific PG:

```bash
ceph pg deep-scrub 1.0
```

Or on a pool:

```bash
ceph osd pool deep-scrub mypool
```

## Unsetting nodeep-scrub

```bash
ceph osd unset nodeep-scrub
```

Deep scrubs resume on their normal schedule after the flag is removed.

## Summary

The `nodeep-scrub` flag is the recommended way to reduce deep scrub I/O without completely stopping data integrity checks. It is best used during recovery operations, heavy I/O periods, or when combined with a scheduled time window. Re-enable deep scrubs promptly to maintain long-term data integrity assurance.
