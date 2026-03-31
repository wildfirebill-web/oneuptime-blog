# How to Set CephFS Subvolume Layouts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CephFS, Layout, Subvolume, Data Pool, Stripe

Description: Configure CephFS subvolume data pool layouts to control where data is stored, enabling different performance profiles for different subvolumes in the same filesystem.

---

CephFS supports multiple data pools per filesystem. By setting a layout on a subvolume, you can direct its data to a specific pool - for example, an SSD-backed pool for low-latency workloads or an erasure-coded pool for archive data. This guide covers configuring layouts at subvolume creation and post-creation.

## Understanding CephFS Layouts

A layout is a set of RADOS object placement attributes that control:
- Which data pool receives the file data
- The stripe unit (object chunk size)
- The stripe count (how many stripes per object set)
- The object size (how large each RADOS object is)

These are set on directory inodes and inherited by files created in that directory.

## Adding a Second Data Pool

First, add a second pool to the filesystem for tiered storage:

```bash
# Create a fast SSD pool
ceph osd pool create cephfs.data.ssd 32
ceph osd pool application enable cephfs.data.ssd cephfs

# Add it as an additional data pool to the filesystem
ceph fs add_data_pool cephfs cephfs.data.ssd

# Verify pools
ceph fs ls
# Look for: data pools: [cephfs.data, cephfs.data.ssd]
```

## Creating a Subvolume with a Custom Data Pool

```bash
# Create a subvolume that stores data in the SSD pool
ceph fs subvolume create cephfs fast-db \
  --data_pool cephfs.data.ssd \
  --size 10737418240

# Create a subvolume using the default pool (no layout override)
ceph fs subvolume create cephfs archive \
  --size 107374182400
```

## Setting Layout on an Existing Subvolume

For an already-mounted subvolume, use `setfattr` to change the layout:

```bash
# Get the subvolume path
SUBVOL_PATH=$(ceph fs subvolume getpath cephfs fast-db)
MOUNT_ROOT="/mnt/cephfs"

# Set the data pool layout
setfattr -n ceph.dir.layout.pool \
  -v cephfs.data.ssd \
  ${MOUNT_ROOT}${SUBVOL_PATH}

# Set stripe unit (1 MB chunks)
setfattr -n ceph.dir.layout.stripe_unit \
  -v 1048576 \
  ${MOUNT_ROOT}${SUBVOL_PATH}

# Verify the layout
getfattr -n ceph.dir.layout ${MOUNT_ROOT}${SUBVOL_PATH}
```

## Configuring Stripe Layout for Large Sequential Files

For workloads with large sequential reads (video, ML training data):

```bash
# Create a subvolume optimized for large sequential I/O
ceph fs subvolume create cephfs media-store \
  --data_pool cephfs.data \
  --size 53687091200

# After mounting, set striping parameters
SUBVOL_PATH=$(ceph fs subvolume getpath cephfs media-store)

# Large stripe unit for sequential workloads
setfattr -n ceph.dir.layout.stripe_unit \
  -v 4194304 \
  ${MOUNT_ROOT}${SUBVOL_PATH}

# Multiple stripes for parallel reads
setfattr -n ceph.dir.layout.stripe_count \
  -v 4 \
  ${MOUNT_ROOT}${SUBVOL_PATH}

# 128 MB RADOS objects
setfattr -n ceph.dir.layout.object_size \
  -v 134217728 \
  ${MOUNT_ROOT}${SUBVOL_PATH}
```

## Verifying Data Pool Placement

After writing data to the subvolume, verify it landed in the correct pool:

```bash
# List objects in the SSD pool
rados -p cephfs.data.ssd ls | head -10

# Get file layout info for a specific file
getfattr -n ceph.file.layout \
  /mnt/tenant-mount/testfile.bin
```

## Summary

CephFS subvolume layouts allow you to direct different subvolumes to different data pools, enabling a tiered storage architecture within a single CephFS deployment. Specify `--data_pool` at subvolume creation time to automatically place data in the desired pool, or adjust layout parameters post-creation via extended attributes. This enables placing latency-sensitive databases on SSD pools while storing archive data on cheaper HDD or erasure-coded pools, all within the same filesystem.
