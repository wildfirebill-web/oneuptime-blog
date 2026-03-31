# How to Understand the deep PG State in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Placement Group, Deep Scrub, State, Storage

Description: Understand the Ceph deep PG state indicating a deep scrub is in progress, how it differs from light scrubbing, and its performance implications.

---

The `deep` PG state (shown as `active+clean+scrubbing+deep`) indicates that a deep scrub is in progress on the PG. Deep scrubs read and verify every byte of every object, providing thorough data integrity validation at a higher I/O cost than light scrubs.

## What deep scrubbing Does

A deep scrub:

1. Reads the complete binary content of every object in the PG
2. Recomputes checksums and compares across all replica OSDs
3. Detects bit rot, corrupted data, and checksum mismatches
4. Updates the `last_deep_scrub_stamp` for the PG

This is more thorough but much more I/O intensive than a light scrub.

## light scrub vs deep scrub

| Aspect | Light Scrub (`scrubbing`) | Deep Scrub (`deep`) |
|--------|--------------------------|---------------------|
| What it reads | Object metadata only | Full object data |
| I/O intensity | Low | High |
| Default frequency | Daily | Weekly |
| Detects bit rot | No | Yes |

## Checking Deep Scrub Status

```bash
ceph pg stat | grep "scrubbing+deep"

# List PGs currently deep scrubbing
ceph pg dump | grep "deep"

# Check last deep scrub time for all PGs
ceph pg dump --format json | jq '.pg_stats[] | {pgid, last_deep_scrub_stamp}'
```

## Deep Scrub Schedule

```bash
# Default deep scrub interval
ceph config get osd osd_deep_scrub_interval
# default: 604800 (7 days)

# Maximum interval before a warning
ceph config get osd osd_scrub_max_interval
```

## Controlling Deep Scrubs

Limit to off-peak hours (shares the same time window as light scrubs):

```bash
ceph config set osd osd_scrub_begin_hour 22
ceph config set osd osd_scrub_end_hour 6
```

Pause all deep scrubs:

```bash
ceph osd set nodeep-scrub
ceph osd unset nodeep-scrub
```

Per-pool:

```bash
ceph osd pool set mypool nodeep-scrub true
ceph osd pool set mypool nodeep-scrub false
```

## Manually Triggering a Deep Scrub

```bash
# Deep scrub a specific PG
ceph pg deep-scrub <pg-id>

# Deep scrub all PGs in a pool
ceph osd pool deep-scrub mypool
```

## Performance Impact of Deep Scrubs

Deep scrubs on HDDs are particularly disruptive. Each OSD in the PG's acting set performs sequential reads across all objects. Monitor impact:

```bash
# Watch OSD IOPS during deep scrub
ceph osd perf | grep -E "apply_latency|commit_latency"

# Check which OSD is deep scrubbing
ceph tell osd.* ops | grep "deep_scrub"
```

Limit concurrent deep scrubs:

```bash
ceph config set osd osd_max_scrubs 1
```

## Deep Scrub Findings

If deep scrub finds errors:

```bash
ceph health detail | grep -i "inconsistent\|deep scrub"
# pg X.Y deep-scrub errors
```

Repair:

```bash
ceph pg repair <pg-id>
```

## Summary

The `deep` PG state indicates thorough byte-by-byte verification of all objects in a PG. It runs weekly by default and is the primary mechanism for detecting silent data corruption (bit rot). While more I/O intensive than light scrubs, regular deep scrubs are essential for long-term data integrity. Control their timing with hour-based windows and the `nodeep-scrub` flag.
