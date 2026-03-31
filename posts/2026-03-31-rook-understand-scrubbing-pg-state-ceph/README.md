# How to Understand the scrubbing PG State in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Placement Group, Scrub, State, Storage

Description: Understand the Ceph scrubbing PG state, how light scrubbing works, its impact on performance, and how to control when it runs.

---

The `scrubbing` PG state indicates that a light scrub is in progress. During a light scrub, the primary OSD compares object metadata and checksums across all replicas without reading the full object data. This is a routine maintenance operation that ensures replica consistency.

## What Light Scrubbing Does

A light scrub:

1. Enumerates all objects in the PG on the primary OSD
2. Requests metadata from all replica OSDs
3. Compares object attributes, sizes, and checksums
4. Reports any discrepancies as `inconsistent`

It does NOT read the full binary content of objects - that is done by deep scrub.

## Checking Scrubbing PGs

```bash
ceph pg stat | grep scrubbing

# List PGs currently scrubbing
ceph pg dump | grep scrubbing

# Confirm scrub is running
ceph status
# active+clean+scrubbing count
```

## Scrubbing Schedule

By default, each PG is scrubbed approximately once per day, within a configurable time window:

```bash
# View current scrub interval settings
ceph config get osd osd_scrub_min_interval
ceph config get osd osd_scrub_max_interval

# Typical defaults
# min_interval: 86400 (1 day)
# max_interval: 604800 (7 days)
```

## Controlling Scrub Timing

Limit scrubbing to off-peak hours:

```bash
ceph config set osd osd_scrub_begin_hour 22
ceph config set osd osd_scrub_end_hour 6
ceph config set osd osd_scrub_begin_week_day 0  # Sunday
ceph config set osd osd_scrub_end_week_day 0    # Sunday
```

## Pausing Scrubs

```bash
# Pause all light scrubs
ceph osd set noscrub

# Resume
ceph osd unset noscrub
```

Per-pool:

```bash
ceph osd pool set mypool noscrub true
ceph osd pool set mypool noscrub false
```

## Manually Triggering a Scrub

```bash
# Scrub a specific PG
ceph pg scrub <pg-id>

# Scrub all PGs in a pool
ceph osd pool scrub mypool
```

## Scrub Impact on Performance

Light scrubs have low to moderate I/O impact:

- They generate metadata read I/O on all replica OSDs
- On busy clusters with many PGs, many concurrent scrubs can add up

Limit concurrent scrubs:

```bash
ceph config set osd osd_max_scrubs 1
```

## Checking Last Scrub Time

```bash
ceph pg dump --format json | jq '.pg_stats[] | {pgid, last_scrub, last_scrub_stamp}'

# Find PGs that haven't been scrubbed recently
ceph pg dump --format json | \
  jq '.pg_stats[] | select(.last_scrub_stamp | . < (now - 86400 | todate)) | .pgid'
```

## What Scrub Finds

If scrub detects inconsistencies:

```bash
ceph health detail | grep inconsistent
# pg X.Y is inconsistent
```

Fix with repair:

```bash
ceph pg repair <pg-id>
```

## Summary

The `scrubbing` state is a routine healthy activity where Ceph verifies object metadata consistency across replicas. Light scrubs run daily by default with low I/O impact. Control their timing with hour-based windows or the `noscrub` flag during peak periods. When scrubs find inconsistencies, use `ceph pg repair` to initiate automatic correction.
