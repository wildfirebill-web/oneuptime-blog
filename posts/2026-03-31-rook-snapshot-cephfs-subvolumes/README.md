# How to Snapshot CephFS Subvolumes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CephFS, Snapshot, Subvolume, Backup, Data Protection

Description: Create, list, and restore CephFS subvolume snapshots to protect tenant data with point-in-time copies that consume minimal additional space.

---

CephFS subvolume snapshots provide point-in-time read-only copies of subvolume data. Snapshots are instantaneous and use copy-on-write semantics, so they consume space only proportional to data changed since the snapshot was taken. This guide covers snapshot creation, access, and restoration.

## How CephFS Snapshots Work

Snapshots are stored in a `.snap` directory inside the subvolume mount point. Each snapshot captures the directory tree state at the moment of creation. Snapshots are stored in the MDS journal and tracked per-inode, making them very space-efficient for incremental changes.

## Creating a Subvolume Snapshot

```bash
# Create a snapshot named "daily-2026-03-31" on subvolume "webapp"
ceph fs subvolume snapshot create cephfs webapp daily-2026-03-31

# Create a snapshot in a specific group
ceph fs subvolume snapshot create cephfs app1 pre-deploy \
  --group_name production
```

## Listing Snapshots

```bash
# List all snapshots for a subvolume
ceph fs subvolume snapshot ls cephfs webapp

# List snapshots in a group subvolume
ceph fs subvolume snapshot ls cephfs app1 --group_name production
```

## Getting Snapshot Information

```bash
# Get info on a specific snapshot
ceph fs subvolume snapshot info cephfs webapp daily-2026-03-31
```

Example output:

```json
{
    "created_at": "2026-03-31T08:00:00.000000",
    "data_pool": "cephfs.data",
    "has_pending_clones": "no",
    "protected": "no",
    "size": 5368709120
}
```

## Accessing Snapshot Data via .snap Directory

When the subvolume is mounted, snapshots are accessible under the `.snap` directory:

```bash
# Mount the subvolume
SUBVOL_PATH=$(ceph fs subvolume getpath cephfs webapp)
mount -t ceph mon1:6789:${SUBVOL_PATH} /mnt/webapp -o name=webapp,secret=<key>

# List available snapshots
ls /mnt/webapp/.snap/

# Access files from a specific snapshot
ls /mnt/webapp/.snap/daily-2026-03-31/

# Restore a file from snapshot
cp /mnt/webapp/.snap/daily-2026-03-31/config.yaml /mnt/webapp/config.yaml
```

## Automating Snapshot Rotation

```bash
#!/bin/bash
# snapshot-rotate.sh - Keep last 7 daily snapshots

FS="cephfs"
SUBVOL="webapp"
DATE=$(date +%Y-%m-%d)
KEEP=7

# Create today's snapshot
ceph fs subvolume snapshot create ${FS} ${SUBVOL} "daily-${DATE}"

# List and sort snapshots, delete oldest if over limit
SNAPSHOTS=$(ceph fs subvolume snapshot ls ${FS} ${SUBVOL} \
  --format json | python3 -c "
import sys, json
snaps = json.load(sys.stdin)
daily = sorted([s['name'] for s in snaps if s['name'].startswith('daily-')])
for s in daily[:-${KEEP}]:
    print(s)
")

for snap in ${SNAPSHOTS}; do
  echo "Removing old snapshot: ${snap}"
  ceph fs subvolume snapshot rm ${FS} ${SUBVOL} ${snap}
done
```

## Protecting Snapshots from Deletion

```bash
# Protect a snapshot (prevents accidental deletion)
ceph fs subvolume snapshot protect cephfs webapp pre-migration

# Remove protection before deletion
ceph fs subvolume snapshot unprotect cephfs webapp pre-migration
```

## Deleting a Snapshot

```bash
# Remove a specific snapshot
ceph fs subvolume snapshot rm cephfs webapp daily-2026-03-30

# Force remove even if protected
ceph fs subvolume snapshot rm cephfs webapp pre-migration --force
```

## Summary

CephFS subvolume snapshots are instantaneous, space-efficient point-in-time copies accessed through the `.snap` directory. Creating snapshots requires a single CLI command and files can be recovered directly by copying from the `.snap` directory without a separate restore process. Automated rotation scripts keep storage costs low by retaining only a rolling window of snapshots. For production systems, protect critical pre-deployment or pre-migration snapshots to prevent accidental removal.
