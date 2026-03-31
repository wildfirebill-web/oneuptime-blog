# How to Understand the recovering PG State in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Placement Group, Recovery, State, Storage

Description: Understand the Ceph recovering PG state, what triggers it, how recovery differs from backfill, and how to monitor and tune recovery speed.

---

The `recovering` PG state indicates that Ceph is actively copying missing or outdated objects from existing replicas to restore the desired redundancy. This state is a positive sign: data is being repaired after a degraded condition.

## What recovering Means

When an OSD comes back after being `down`, its copy of PG data may be stale (missed writes while it was offline). Ceph must update these objects before the OSD can fully rejoin as an active replica. This process is called recovery.

During recovery, the PG is typically `active+recovering` - it continues serving I/O while repair happens in the background.

## Recovery vs Backfill

| Process | Trigger | What moves |
|---------|---------|------------|
| Recovery | OSD came back after being down | Only changed objects |
| Backfill | New OSD added or CRUSH change | All PG objects |

Recovery is usually faster than backfill because only changed objects need to transfer.

## Checking Recovery Progress

```bash
ceph status
# recovering 512/3072 objects, 200 MiB/s

ceph pg stat
# active+recovering count

# Watch live progress
watch -n2 ceph status
```

For per-PG recovery details:

```bash
ceph pg dump | grep recovering
ceph pg <pg-id> query | jq '.recovery_state'
```

## Recovery Rate

Check the current recovery throughput:

```bash
ceph status --format json | jq '.pgmap | {recovering_bytes_per_sec, recovering_objects_per_sec}'
```

## Tuning Recovery Speed

Recovery rate is controlled by several parameters:

```bash
# Max concurrent recovery operations per OSD
ceph config set osd osd_recovery_max_active_hdd 1
ceph config set osd osd_recovery_max_active_ssd 3

# Priority relative to client I/O (higher = more priority)
ceph config set osd osd_recovery_op_priority 3

# Max bytes per recovery operation
ceph config set osd osd_recovery_max_chunk 8388608  # 8 MiB
```

Increase these values to speed up recovery at the cost of client I/O:

```bash
ceph config set osd osd_recovery_max_active_hdd 3
ceph config set osd osd_recovery_op_priority 15
```

## What Happens After Recovery

When all objects in a PG have been recovered:

1. The PG transitions from `active+recovering` to `active+clean`
2. The recovered OSD becomes a full peer for the PG

Watch for this transition:

```bash
watch "ceph pg stat | grep -E 'recovering|active\+clean'"
```

## Identifying Stuck Recovery

If recovery seems to have stalled:

```bash
# Check for PGs stuck in recovering
ceph health detail | grep -i "stuck"

# Check OSD logs for recovery errors
journalctl -u ceph-osd@3 | grep -i "recovery\|error"

# Force a PG to re-peer
ceph pg <pg-id> mark_unfound_lost revert
```

## Summary

The `recovering` state shows Ceph is actively restoring replica redundancy after an OSD failure and return. It is a healthy transitional state that progresses toward `active+clean`. Tune `osd_recovery_max_active` and `osd_recovery_op_priority` to balance recovery speed against client I/O impact based on your operational priorities.
