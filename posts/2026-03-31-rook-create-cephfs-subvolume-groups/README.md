# How to Create CephFS Subvolume Groups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CephFS, Subvolume, Group, Filesystem, Storage

Description: Organize CephFS subvolumes into groups to apply shared pool layouts and manage related subvolumes as a unit for multi-tenant or multi-team storage.

---

CephFS subvolume groups are containers that organize subvolumes with a shared configuration, such as a common data pool layout. Groups allow you to manage related subvolumes together - for example, all subvolumes for a production team or a particular application namespace.

## What is a Subvolume Group

A subvolume group:
- Acts as a namespace/directory above individual subvolumes
- Allows setting default pool layouts that apply to all subvolumes in the group
- Makes it easier to list, snapshot, and manage related subvolumes
- Corresponds to a directory under `/volumes/` in the CephFS root

The default group is `_nogroup` - subvolumes created without specifying a group go here.

## Creating a Subvolume Group

```bash
# Create a group named "production"
ceph fs subvolumegroup create cephfs production

# Create with a specific data pool layout
ceph fs subvolumegroup create cephfs analytics \
  --pool_layout cephfs.data.ssd

# Create with a size limit (applies quota to the group directory)
ceph fs subvolumegroup create cephfs dev-team \
  --size 107374182400   # 100 GB

# Create with specific UID/GID ownership
ceph fs subvolumegroup create cephfs ml-workloads \
  --uid 2000 \
  --gid 2000 \
  --mode 0750
```

## Listing Subvolume Groups

```bash
# List all groups in the filesystem
ceph fs subvolumegroup ls cephfs

# Get info on a specific group
ceph fs subvolumegroup info cephfs production
```

## Creating Subvolumes Inside a Group

```bash
# Create subvolumes within the production group
ceph fs subvolume create cephfs app1 --group_name production --size 21474836480
ceph fs subvolume create cephfs app2 --group_name production --size 10737418240
ceph fs subvolume create cephfs shared-config --group_name production --size 1073741824

# List subvolumes in the group
ceph fs subvolume ls cephfs --group_name production
```

## Getting Paths for Group Subvolumes

```bash
# Get path for a subvolume in a group
ceph fs subvolume getpath cephfs app1 --group_name production
# Output: /volumes/production/app1/<uuid>
```

## Snapshotting an Entire Group

You can create snapshots at the subvolume group level to capture all subvolumes at once:

```bash
# Create a group snapshot
ceph fs subvolumegroup snapshot create cephfs production daily-backup

# List group snapshots
ceph fs subvolumegroup snapshot ls cephfs production

# Remove a group snapshot
ceph fs subvolumegroup snapshot rm cephfs production daily-backup
```

## Example: Setting Up a Multi-Team Structure

```bash
# Create teams
for team in frontend backend data-platform; do
  ceph fs subvolumegroup create cephfs ${team}
done

# Create per-team subvolumes
ceph fs subvolume create cephfs assets --group_name frontend --size 10737418240
ceph fs subvolume create cephfs api-cache --group_name backend --size 5368709120
ceph fs subvolume create cephfs datasets --group_name data-platform --size 107374182400

# Verify structure
for team in frontend backend data-platform; do
  echo "=== ${team} ==="
  ceph fs subvolume ls cephfs --group_name ${team}
done
```

## Summary

CephFS subvolume groups provide a layer of organization above individual subvolumes, making it easier to manage multi-team or multi-application storage within a single CephFS filesystem. Groups can have default pool layouts, size limits, and ownership settings that apply to all contained subvolumes. Combined with group-level snapshots, they offer a practical way to organize and protect related filesystem namespaces.
