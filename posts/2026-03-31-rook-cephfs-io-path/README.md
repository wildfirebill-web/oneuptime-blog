# How to Understand the CephFS IO Path

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Performance, Architecture

Description: Learn how the CephFS I/O path works from client to OSD, including metadata lookup, capability acquisition, and direct data I/O to RADOS in Rook-Ceph.

---

## Overview

Understanding the CephFS I/O path is essential for diagnosing performance issues and designing workloads that maximize throughput. Unlike NFS, CephFS clients communicate directly with storage daemons (OSDs) for data I/O after obtaining authorization from the MDS. This architecture enables high parallel throughput and eliminates the server-side bottleneck common in traditional NAS systems.

## Read I/O Path

When a CephFS client reads a file, the following steps occur:

```text
1. Client -> MDS: Lookup file path (if not cached)
   MDS returns inode metadata + read capability

2. Client -> MDS: Request Rd cap for the file
   MDS grants Rd capability if no conflicting writers

3. Client computes RADOS object locations:
   object_id = (file_ino << 32) | (file_offset / object_size)
   pool = file layout pool

4. Client -> OSD(s): Direct RADOS read for each object
   No MDS involvement for data I/O

5. Client returns data to application
```

## Write I/O Path

```text
1. Client -> MDS: Request Wr (write) capability
   MDS may revoke read caps from other clients first

2. Client buffers writes in page cache
   (or uses direct I/O for O_DIRECT)

3. Client writes data directly to OSD(s):
   Client -> OSD Primary -> OSD Replicas

4. Client -> MDS: Update file size/mtime after flush
   (or on close, or periodically for long-running writes)
```

## Capability System

The capability (cap) system is what makes the CephFS I/O path efficient. Clients hold caps that grant them permission to perform operations locally without consulting the MDS for every I/O:

```text
Cap Types:
- Rd    - Read file data
- Wr    - Write file data
- As    - Read/write file attributes
- Lc    - Link/unlink in directory
- Xs    - Extended attributes
- Sn    - Snapshot operations
```

## Observe the I/O Path in Action

Monitor RADOS operations for a file being written:

```bash
# Watch OSD operation rate during writes
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool stats cephfs-data0

# Watch MDS cap operations
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph tell mds.cephfs:0 perf dump | jq '.mds | {cap_revoke, cap_grant}'
```

## Object Mapping

To find which RADOS object stores a specific byte range of a file:

```bash
# Find the RADOS object for byte offset 0 of file with inode 100
# object_name = <inode_hex>.<object_offset_hex>
python3 -c "
inode = 100
offset = 0
object_size = 4 * 1024 * 1024  # 4MB default
obj_offset = offset // object_size
print(f'{inode:016x}.{obj_offset:08x}')
"
```

## Data Locality and Read Affinity

CephFS clients can read from the nearest OSD replica using read affinity:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client osd_read_from_replica balance
```

## Summary

The CephFS I/O path is a split-plane architecture: the MDS handles metadata and capability management while clients communicate directly with OSDs for all data I/O. This design eliminates metadata-server data bottlenecks and enables clients to achieve full OSD throughput in parallel. Understanding capability acquisition, object mapping, and the direct OSD I/O flow is essential for diagnosing latency and throughput issues in Rook-Ceph CephFS deployments.
