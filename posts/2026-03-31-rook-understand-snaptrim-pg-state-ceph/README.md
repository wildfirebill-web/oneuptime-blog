# How to Understand the snaptrim PG State in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Placement Group, Snapshot, State, Storage

Description: Understand the Ceph snaptrim PG state, why it occurs after snapshot deletion, how it impacts performance, and how to manage snapshot trimming.

---

The `snaptrim` PG state indicates that Ceph is trimming (removing) snapshot data after a snapshot has been deleted. This is a background cleanup operation that can be resource-intensive when many snapshots are deleted simultaneously.

## What snaptrim Does

When you delete a Ceph snapshot, the data is not immediately freed. Instead, Ceph must walk through every object in the PG, identify object versions that were part of the deleted snapshot, and remove them. This process is called snapshot trimming.

Steps during snaptrim:
1. The PG receives the snapshot deletion command
2. It enters `active+clean+snaptrim` state
3. Objects with snapshot data from the deleted snapshot are processed
4. Old snapshot versions are removed from each object
5. PG returns to `active+clean` when complete

## When snaptrim Appears

```bash
# Delete an RBD snapshot
rbd snap rm mypool/myimage@snapshot1

# The PG(s) hosting that image's objects will enter snaptrim
ceph pg stat | grep snaptrim

# Watch it progress
watch ceph pg stat
```

## Checking snaptrim Status

```bash
ceph pg stat | grep snaptrim

# List PGs currently trimming
ceph pg dump | grep snaptrim

# Count queued snaptrim operations
ceph health detail | grep snaptrim
```

## snaptrim_wait State

When too many PGs are already trimming, additional ones queue as `snaptrim_wait`:

```bash
ceph pg dump | grep "snaptrim_wait"
```

## Performance Impact

Snaptrim reads and modifies many objects. On pools with large images and many snapshots, trimming can:

- Generate significant I/O on OSD disks
- Cause latency spikes for client I/O
- Run for minutes to hours for large images

## Controlling snaptrim Speed

```bash
# Limit snaptrim I/O priority
ceph config set osd osd_snap_trim_priority 5  # lower = less priority

# Delay between snaptrim operations (microseconds)
ceph config set osd osd_snap_trim_sleep_hdd 100000   # 100ms per object
ceph config set osd osd_snap_trim_sleep_ssd 0

# Max concurrent snaptrim per OSD
ceph config set osd osd_snap_trim_max 10
```

## Pausing Snaptrim

```bash
# Set flag to pause all snaptrim
ceph osd set notrim

# Verify
ceph osd dump | grep flags

# Resume
ceph osd unset notrim
```

Note: `notrim` also pauses other trim operations.

## Stale snaptrim PGs

If PGs stay in snaptrim for a very long time:

```bash
ceph health detail | grep "stuck snaptrimming"

# Check OSD logs for snaptrim activity
journalctl -u ceph-osd@0 | grep -i "snaptrim"

# Force snaptrim to continue
ceph pg <pg-id> snaptrim
```

## Preventing snaptrim Buildup

```bash
# Delete old snapshots regularly, one at a time
rbd snap purge mypool/myimage  # removes all snapshots

# Monitor snaptrim queue length
ceph pg dump --format json | jq '[.pg_stats[] | select(.state | contains("snaptrim"))] | length'
```

## Summary

The `snaptrim` PG state is a normal background operation triggered by snapshot deletion. It performs cleanup work to free the storage space used by deleted snapshots. For large images with many snapshots, snaptrim can run for extended periods and consume significant I/O. Control its impact with `osd_snap_trim_sleep_hdd` to rate-limit the cleanup, or use the `notrim` flag to pause it during critical workload periods.
