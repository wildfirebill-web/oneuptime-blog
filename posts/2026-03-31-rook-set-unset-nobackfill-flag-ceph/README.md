# How to Set and Unset the nobackfill Flag in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, OSD, Backfill, Flag, Maintenance

Description: Learn how the Ceph nobackfill flag pauses PG backfilling and when to use it to prevent rebalancing I/O from impacting cluster performance.

---

Backfilling in Ceph is the process of migrating PG data to new or replacement OSDs. The `nobackfill` flag pauses this migration, giving you control over when the associated I/O overhead occurs.

## What Backfilling Is

Backfill occurs when:

- New OSDs are added to the cluster and the CRUSH map routes PGs to them
- An OSD is replaced and the new one needs its PG copies
- CRUSH map changes cause PG remapping

During backfill, the source OSD copies entire PG contents to the destination OSD. This is more expensive than recovery because all objects must be transferred, not just changed ones.

## Backfill vs Recovery

| Process | Trigger | Data Transferred |
|---------|---------|------------------|
| Recovery | OSD came back after being down | Only changed objects |
| Backfill | New OSD or CRUSH change | All objects in the PG |

## Setting nobackfill

```bash
ceph osd set nobackfill
```

Verify:

```bash
ceph osd dump | grep flags
ceph status
# HEALTH_WARN: nobackfill flag(s) set
```

Existing `backfilling` PGs will pause at their current state.

## Use Case - Staged OSD Addition

When adding many OSDs at once, you may want to control the rebalancing pace:

```bash
# Pause backfill before adding OSDs
ceph osd set nobackfill

# Add new OSDs via Rook or cephadm
ceph orch apply osd --all-available-devices

# OSDs are added but no data moves yet
ceph osd tree

# During a maintenance window, allow backfill
ceph osd unset nobackfill
```

## Use Case - Protecting Production During Capacity Expansion

```bash
# Expand cluster during off-hours
ceph osd set nobackfill

# Verify new OSD connectivity
ceph osd stat

# Allow backfill overnight
ceph osd unset nobackfill
```

## Tuning Backfill Speed

Instead of a full stop, you can throttle backfill:

```bash
# Limit concurrent backfill operations
ceph config set osd osd_max_backfills 1

# Lower backfill priority vs client I/O
ceph config set osd osd_backfill_scan_min 4
ceph config set osd osd_backfill_scan_max 32
```

## Monitoring Backfill Progress

```bash
ceph status
# Look for: "X/Y objects backfilling in Z PGs"

ceph pg stat
# active+remapped+backfilling count

# Estimate completion time
ceph pg dump | grep backfilling
```

## Unsetting nobackfill

```bash
ceph osd unset nobackfill
```

Backfilling resumes from where it paused. Monitor progress:

```bash
watch ceph status
```

## Summary

The `nobackfill` flag pauses PG migration to new or replacement OSDs, protecting client I/O during cluster expansion or CRUSH changes. Use it to schedule rebalancing during off-peak hours, and always combine it with `norecover` and `noout` when you need comprehensive control over data movement during planned maintenance.
