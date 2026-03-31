# How to Set Snapshot Retention Policies in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Snapshot, Retention, Data Protection, Lifecycle

Description: Learn how to configure snapshot retention policies in Ceph for CephFS and RBD images to automatically prune old snapshots and manage storage consumption over time.

---

## Why Set Retention Policies?

Without retention policies, automatic snapshot schedules create an ever-growing list of snapshots that consume increasing amounts of storage. Retention policies tell Ceph how many snapshots to keep per time period, automatically deleting older ones.

## CephFS Snapshot Retention

### Adding Retention to an Existing Schedule

```bash
# Keep 7 daily snapshots
ceph fs snap-schedule retention add /data d 7 --fs myfs

# Keep 4 weekly snapshots
ceph fs snap-schedule retention add /data w 4 --fs myfs

# Keep 12 monthly snapshots
ceph fs snap-schedule retention add /data M 12 --fs myfs

# Keep 24 hourly snapshots
ceph fs snap-schedule retention add /data h 24 --fs myfs
```

### Retention Period Codes

| Code | Period |
|------|--------|
| m | Minute |
| h | Hour |
| d | Day |
| w | Week |
| M | Month |
| y | Year |

### Viewing Retention Settings

```bash
ceph fs snap-schedule status /data --fs myfs
```

Look for the `retention` field in the output:
```json
{
  "retention": {"d": 7, "w": 4, "M": 12}
}
```

### Removing a Retention Rule

```bash
ceph fs snap-schedule retention rm /data d --fs myfs
```

## RBD Snapshot Retention

RBD snap schedule manages retention via the `--retain-count` option:

```bash
# Keep only the last 5 snapshots
rbd snap schedule add --pool mypool --image myimage 1d --retain 5
```

For images using the snap schedule module:

```bash
rbd snap schedule ls --pool mypool --image myimage --format json
```

## Custom Retention Script for RBD

For pools without built-in retention, use a script:

```bash
#!/bin/bash
POOL="mypool"
IMAGE="myimage"
RETENTION=7

SNAPS=$(rbd snap ls $POOL/$IMAGE --format json | jq -r '.[].name' | sort)
SNAP_COUNT=$(echo "$SNAPS" | wc -l)

if [ "$SNAP_COUNT" -gt "$RETENTION" ]; then
  DELETE_COUNT=$((SNAP_COUNT - RETENTION))
  echo "$SNAPS" | head -n "$DELETE_COUNT" | while read snap; do
    echo "Removing snapshot: $snap"
    rbd snap rm "$POOL/$IMAGE@$snap" 2>/dev/null || \
      echo "Cannot remove $snap (may be protected or have children)"
  done
fi
```

## Implementing a 3-2-1 Snapshot Retention Policy

A common backup strategy keeps 3 copies in 2 formats over 1 off-site location:

```bash
# 24 hourly snapshots (1 day coverage with hourly granularity)
ceph fs snap-schedule retention add /data h 24 --fs myfs

# 7 daily snapshots (1 week coverage)
ceph fs snap-schedule retention add /data d 7 --fs myfs

# 4 weekly snapshots (1 month coverage)
ceph fs snap-schedule retention add /data w 4 --fs myfs

# 12 monthly snapshots (1 year coverage)
ceph fs snap-schedule retention add /data M 12 --fs myfs
```

## Monitoring Snapshot Count and Storage

```bash
# Count snapshots per directory
ls /mnt/cephfs/data/.snap/ | wc -l

# Storage used by snapshots (approximate)
ceph df detail | grep myfs
```

## Summary

Snapshot retention policies in Ceph are set using `ceph fs snap-schedule retention add` for CephFS directories, specifying the time period code and count. For RBD, use the `--retain` flag with `rbd snap schedule add` or implement a custom pruning script. Implement a graduated retention strategy with more frequent short-term snapshots and fewer long-term ones, matching retention counts to your recovery time and recovery point objectives.
