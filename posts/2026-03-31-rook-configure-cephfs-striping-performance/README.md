# How to Configure CephFS Striping for Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Performance, Striping

Description: Configure CephFS file striping in Rook to spread data across multiple OSDs, increasing throughput for parallel workloads and large file access patterns.

---

## What is CephFS Striping

CephFS striping splits a single file's data across multiple RADOS objects, each stored on a different OSD. This enables parallel I/O to multiple OSDs simultaneously, increasing aggregate throughput beyond what any single OSD can deliver. Three parameters control striping behavior: `stripe_unit`, `stripe_count`, and `object_size`.

- `stripe_unit`: the size of each stripe chunk (default: 4 MB)
- `stripe_count`: how many OSDs a file is striped across
- `object_size`: the total size of each RADOS object (must be a multiple of `stripe_unit * stripe_count`)

## Setting Default Striping at the Filesystem Level

Configure global defaults for all new files:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs set cephfs default_stripe_unit 4194304

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs set cephfs default_stripe_count 8
```

## Setting Striping on Specific Directories

Override defaults for specific workload directories using xattrs on a kernel-mounted CephFS:

```bash
# Mount CephFS first
mount -t ceph mon-ip:/ /mnt/cephfs -o name=admin,secret=<key>

# Set layout on the target directory
setfattr -n ceph.dir.layout.stripe_unit -v 1048576 /mnt/cephfs/hpc-jobs
setfattr -n ceph.dir.layout.stripe_count -v 16 /mnt/cephfs/hpc-jobs
setfattr -n ceph.dir.layout.object_size -v 16777216 /mnt/cephfs/hpc-jobs
setfattr -n ceph.dir.layout.pool -v cephfs-data0 /mnt/cephfs/hpc-jobs
```

Verify the layout:

```bash
getfattr -n ceph.dir.layout /mnt/cephfs/hpc-jobs
```

## Per-File Striping

For already-created files, you can set layout before writing using `ceph-fuse` or the admin socket:

```bash
# Using ceph admin CLI on a newly created empty file
setfattr -n ceph.file.layout.stripe_unit -v 2097152 /mnt/cephfs/hpc-jobs/newfile
setfattr -n ceph.file.layout.stripe_count -v 8 /mnt/cephfs/hpc-jobs/newfile
```

Note: File layout can only be changed on empty files.

## Choosing Stripe Parameters

| Workload | stripe_unit | stripe_count | object_size |
|----------|-------------|--------------|-------------|
| HPC/parallel | 1 MB | 16 | 16 MB |
| Video streaming | 4 MB | 8 | 32 MB |
| Database | 256 KB | 4 | 1 MB |
| General | 4 MB | 4 | 16 MB |

Larger stripe_count increases throughput but also increases the number of OSD operations per I/O, adding latency for small files.

## Verifying Striping Effect with fio

Test parallel write performance with striping:

```bash
fio --name=stripe-test --ioengine=posixaio --rw=write \
    --bs=4m --size=20g --iodepth=32 --numjobs=8 \
    --group_reporting \
    --directory=/mnt/cephfs/hpc-jobs/
```

Monitor OSD utilization balance during the test:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd perf
```

All OSDs should show similar throughput if striping is distributing load evenly.

## Summary

CephFS striping dramatically improves throughput for large files by distributing data across many OSDs simultaneously. Setting stripe parameters at the directory level via xattrs allows different workloads to use optimal layouts. Always validate that object_size equals `stripe_unit * stripe_count` to avoid internal fragmentation.
