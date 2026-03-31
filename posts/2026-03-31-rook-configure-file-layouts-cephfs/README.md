# How to Configure File Layouts in CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Layout, Storage

Description: Learn how to configure CephFS file layouts to control stripe width, count, object size, and pool placement for optimized storage performance in Rook-Ceph.

---

## Overview

CephFS file layouts control how file data is distributed across RADOS objects and stored in data pools. By configuring the layout, you can optimize for different workload patterns - large sequential files benefit from wider stripes, while small random-access files may perform better with smaller objects. Layouts can be set per-file or inherited from directory defaults.

## Layout Parameters

A CephFS layout consists of four parameters:

```text
stripe_unit    - Size in bytes of each stripe unit (default: 4MB)
stripe_count   - Number of objects data is striped across (default: 1)
object_size    - Maximum size in bytes of each RADOS object (default: 4MB)
pool           - The RADOS data pool where file data is stored
```

## View the Current Layout

Check the default layout of the filesystem root:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph tell mds.cephfs:0 get default_file_layout
```

Check the layout of a specific file or directory:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c \
  "getfattr -n ceph.file.layout /mnt/cephfs/myfile"
```

## Set a Directory Default Layout

Set the layout on a directory so all new files inherit it:

```bash
# Set stripe_count to 4 for parallel access
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c \
  "setfattr -n ceph.dir.layout.stripe_count -v 4 /mnt/cephfs/bigfiles"

# Set larger object size for large files
kubectl -n rook-ceph exec -it deploy/rook-cephtools -- bash -c \
  "setfattr -n ceph.dir.layout.object_size -v 67108864 /mnt/cephfs/bigfiles"

# Store data in a specific pool
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c \
  "setfattr -n ceph.dir.layout.pool -v cephfs-ssd-data /mnt/cephfs/hotdata"
```

## Set a File Layout

File layouts must be set before writing any data to the file:

```bash
# Create an empty file
touch /mnt/cephfs/largefile.dat

# Set layout before writing
setfattr -n ceph.file.layout.stripe_count -v 8 /mnt/cephfs/largefile.dat
setfattr -n ceph.file.layout.stripe_unit -v 4194304 /mnt/cephfs/largefile.dat
setfattr -n ceph.file.layout.object_size -v 134217728 /mnt/cephfs/largefile.dat

# Now write data
dd if=/dev/zero of=/mnt/cephfs/largefile.dat bs=1G count=10
```

## Layout for High-Throughput Workloads

For large parallel I/O (HPC, analytics), use high stripe counts:

```bash
setfattr -n ceph.dir.layout.stripe_count -v 16 /mnt/cephfs/hpc
setfattr -n ceph.dir.layout.stripe_unit -v 4194304 /mnt/cephfs/hpc
setfattr -n ceph.dir.layout.object_size -v 67108864 /mnt/cephfs/hpc
```

## Layout for Small Files

For workloads with many small files, reduce object size:

```bash
setfattr -n ceph.dir.layout.object_size -v 4194304 /mnt/cephfs/smallfiles
setfattr -n ceph.dir.layout.stripe_count -v 1 /mnt/cephfs/smallfiles
```

## Summary

CephFS file layouts provide fine-grained control over how data is distributed across RADOS objects and pools. By setting `stripe_count`, `stripe_unit`, `object_size`, and `pool` on directories or files using extended attributes, you can tune storage performance for specific workload patterns. In Rook-Ceph deployments, combining multiple data pools with tailored layouts enables tiered storage where different application datasets are optimally placed on SSDs or HDDs.
