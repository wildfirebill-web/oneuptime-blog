# How to Delete CephFS Subvolumes and Groups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CephFS, Subvolume, Deletion, Cleanup, Storage

Description: Safely delete CephFS subvolumes and subvolume groups, handling snapshot dependencies and retained state to fully reclaim storage space.

---

Deleting CephFS subvolumes is not always straightforward - snapshots, clones, and retained state can prevent immediate deletion. This guide covers the complete deletion workflow including how to handle these dependencies.

## Deletion States

When you delete a subvolume, it may not be immediately purged. Ceph uses these states:

- **complete** - normal active state
- **purging** - deletion in progress, being removed from RADOS
- **retained** - deletion was issued but snapshots still exist; the subvolume is removed from the listing but the data remains

Understanding these states prevents confusion when storage is not immediately reclaimed.

## Deleting a Simple Subvolume

```bash
# Delete a subvolume with no snapshots
ceph fs subvolume rm cephfs webapp

# Delete a subvolume in a group
ceph fs subvolume rm cephfs app1 --group_name production

# Force delete even if the subvolume is in an unexpected state
ceph fs subvolume rm cephfs webapp --force
```

## Deleting a Subvolume with Snapshots

If the subvolume has snapshots, delete them first:

```bash
# List snapshots
ceph fs subvolume snapshot ls cephfs webapp

# Delete each snapshot
ceph fs subvolume snapshot rm cephfs webapp daily-2026-03-30
ceph fs subvolume snapshot rm cephfs webapp daily-2026-03-31

# Now delete the subvolume
ceph fs subvolume rm cephfs webapp
```

Or use the `--retain-snapshots` flag to delete the subvolume but keep the data accessible via snapshots:

```bash
# Marks subvolume as deleted but retains snapshot data
ceph fs subvolume rm cephfs webapp --retain-snapshots
# State becomes "retained"
```

## Cleaning Up Retained Subvolumes

A retained subvolume still occupies space. To fully purge it, delete all its snapshots:

```bash
# List retained subvolumes (they appear in ls output with state "retained")
ceph fs subvolume ls cephfs --format json | python3 -c "
import sys, json
for sv in json.load(sys.stdin):
    info = sv.get('info', {})
    if info.get('state') == 'retained':
        print(sv['name'])
"

# Remove remaining snapshots on a retained subvolume
ceph fs subvolume snapshot ls cephfs webapp
ceph fs subvolume snapshot rm cephfs webapp pre-migration
# After last snapshot removed, purge completes automatically
```

## Deleting a Subvolume Group

You can only delete a group after all its subvolumes are removed:

```bash
# List subvolumes in the group
ceph fs subvolume ls cephfs --group_name dev-team

# Delete each subvolume
ceph fs subvolume rm cephfs feature-branch --group_name dev-team
ceph fs subvolume rm cephfs sandbox --group_name dev-team

# Delete the group
ceph fs subvolumegroup rm cephfs dev-team

# Force delete (skips the subvolume check - use with caution)
ceph fs subvolumegroup rm cephfs dev-team --force
```

## Bulk Cleanup Script

```bash
#!/bin/bash
# cleanup-group.sh - Remove all subvolumes and snapshots in a group

FS="cephfs"
GROUP="$1"

if [ -z "${GROUP}" ]; then
  echo "Usage: $0 <group-name>"
  exit 1
fi

echo "Cleaning up group: ${GROUP}"

for sv in $(ceph fs subvolume ls ${FS} --group_name ${GROUP} \
  --format json | python3 -c "import sys,json; [print(s['name']) for s in json.load(sys.stdin)]"); do

  echo "  Processing subvolume: ${sv}"

  # Delete all snapshots
  for snap in $(ceph fs subvolume snapshot ls ${FS} ${sv} \
    --group_name ${GROUP} --format json | \
    python3 -c "import sys,json; [print(s['name']) for s in json.load(sys.stdin)]"); do
    echo "    Removing snapshot: ${snap}"
    ceph fs subvolume snapshot rm ${FS} ${sv} ${snap} --group_name ${GROUP}
  done

  # Remove the subvolume
  ceph fs subvolume rm ${FS} ${sv} --group_name ${GROUP} --force
  echo "  Deleted: ${sv}"
done

# Remove the group
ceph fs subvolumegroup rm ${FS} ${GROUP}
echo "Group ${GROUP} deleted."
```

## Summary

Deleting CephFS subvolumes requires handling snapshot dependencies before the underlying data is purged. Subvolumes with remaining snapshots enter a "retained" state and must have their snapshots removed before storage is fully reclaimed. Always list and remove snapshots explicitly before deleting subvolumes in production, or use `--retain-snapshots` when you want to preserve historical data after retiring a subvolume. Groups can only be deleted after all contained subvolumes are removed.
