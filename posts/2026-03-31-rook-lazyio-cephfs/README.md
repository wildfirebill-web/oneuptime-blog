# How to Use LazyIO in CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Performance, IO

Description: Learn how to use LazyIO in CephFS to improve performance for applications that can tolerate relaxed consistency guarantees in Rook-Ceph deployments.

---

## Overview

LazyIO is a CephFS feature that allows clients to aggressively cache file data and metadata locally, deferring consistency enforcement until explicitly requested. By relaxing the strict close-to-open (CAO) consistency model, LazyIO enables significantly higher I/O throughput for applications that can tolerate reading slightly stale data from other clients - such as HPC applications using checkpoint files or parallel read-heavy workloads.

## How LazyIO Differs from Default CAO Consistency

Default CephFS behavior (close-to-open):
- Data written by Client A becomes visible to Client B after Client A closes the file
- MDS enforces capability revocation when multiple clients access the same file

LazyIO behavior:
- Clients hold read/write capabilities even when other clients are accessing the file
- Data may not be immediately visible to other clients
- Application is responsible for explicit synchronization when needed

## When to Use LazyIO

LazyIO is appropriate for:
- HPC workloads where different processes write independent sections of a file
- Parallel scientific computing with checkpoint/restart patterns
- Read-heavy workloads where multiple clients read the same large files
- Applications that manage their own consistency via custom synchronization

## Enable LazyIO via FUSE Client

```bash
# Mount with LazyIO enabled
ceph-fuse /mnt/cephfs --client_cache_size=65536 --fuse_big_writes=true

# Enable LazyIO via config
cat >> /etc/ceph/ceph.conf <<EOF
[client]
client_cache_size = 65536
client_readahead_max_bytes = 33554432
EOF
```

## Enable LazyIO Programmatically (libcephfs)

For applications using the libcephfs API directly:

```c
#include <cephfs/libcephfs.h>

struct ceph_mount_info *cmount;
ceph_create(&cmount, "admin");
ceph_conf_read_file(cmount, "/etc/ceph/ceph.conf");
ceph_mount(cmount, "/");

// Open a file and enable LazyIO
int fd = ceph_open(cmount, "/myfile", O_RDWR | O_CREAT, 0644);
ceph_lazyio(cmount, fd, 1);  /* 1 = enable LazyIO */

/* Write without triggering cap revocation */
ceph_write(cmount, fd, buf, len, 0);

/* Explicitly sync when other clients need to see data */
ceph_lazyio_propagate(cmount, fd, 0, 0);
ceph_lazyio_synchronize(cmount, fd, 0, 0);
```

## Explicit Synchronization with LazyIO

When you need to make writes visible to other clients:

```c
/* Propagate local writes to OSD */
ceph_lazyio_propagate(cmount, fd, offset, count);

/* Ensure remote writes are visible locally */
ceph_lazyio_synchronize(cmount, fd, offset, count);
```

## Measure LazyIO Performance Benefit

Compare throughput with and without LazyIO using `fio`:

```bash
# Without LazyIO
fio --name=baseline --rw=randwrite --bs=4k --size=1G \
    --numjobs=8 --ioengine=sync --filename=/mnt/cephfs/test

# With LazyIO (higher client cache)
fio --name=lazyio --rw=randwrite --bs=4k --size=1G \
    --numjobs=8 --ioengine=sync --filename=/mnt/cephfs/test \
    --rw_sequencer=0
```

## Limitations

- LazyIO is not suitable for shared databases or any application requiring strict consistency
- Files using LazyIO must not be concurrently written by multiple clients without explicit synchronization
- LazyIO is primarily available through the libcephfs API; FUSE client support is limited

## Summary

LazyIO in CephFS provides a relaxed consistency mode that can dramatically improve I/O throughput for applications that manage their own consistency. By holding capabilities longer and caching more aggressively, LazyIO reduces MDS coordination overhead. In Rook-Ceph deployments, LazyIO is most valuable for HPC and parallel computing workloads where independent writers access non-overlapping file regions and explicit synchronization is used at synchronization points.
