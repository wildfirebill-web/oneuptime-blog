# How to Configure Directory Fragmentation in CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, MDS, Scalability

Description: Learn how CephFS directory fragmentation works and how to configure it to handle very large directories efficiently in Rook-Ceph deployments.

---

## Overview

CephFS implements directory fragmentation to handle directories with very large numbers of entries (tens of thousands or more). Without fragmentation, a single large directory becomes a metadata bottleneck as all entries must be stored in a single RADOS object. Directory fragmentation splits large directories into multiple fragments distributed across RADOS objects, enabling horizontal scaling of metadata for huge directories.

## How Directory Fragmentation Works

When a directory grows beyond a configured threshold, the MDS fragments it by:

1. Splitting the directory's hash space into multiple ranges
2. Storing each range in a separate RADOS object
3. Using a consistent hash of the filename to determine which fragment it belongs to

This process is transparent to clients - they see a normal directory view regardless of how many fragments it has been split into.

## Check If Fragmentation is Enabled

Verify that directory fragmentation is enabled on the filesystem:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs get cephfs | grep "allow_dirfrags"
```

## Enable Directory Fragmentation

Fragmentation should be enabled by default in modern Ceph versions. If disabled:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs set cephfs allow_dirfrags true
```

## Configure Fragmentation Threshold

Set the number of entries at which a directory will be split:

```bash
# Default is 10,000 entries
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_bal_split_size 10000

# Number of entries at which to merge fragments back together
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_bal_merge_size 5000
```

## View Directory Fragment State

Check how many fragments a directory has using the MDS tell interface:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph tell mds.cephfs:0 dirfrag ls /large-directory
```

## Force Immediate Fragmentation

To pre-fragment a directory before it grows large (proactive fragmentation):

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph tell mds.cephfs:0 dirfrag split /large-directory 0/1 3
```

The last argument is the split bits (2^3 = 8 fragments).

## Check Directory Entry Count

Monitor how many entries are in a directory to anticipate fragmentation:

```bash
# Count entries (may be slow for very large directories)
ls -la /mnt/cephfs/large-directory | wc -l

# Or use the MDS stat
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph tell mds.cephfs:0 dump_dir /large-directory
```

## Performance Considerations

For directories expected to contain millions of entries:
- Pre-fragment the directory using `dirfrag split` before populating it
- Use a higher split threshold for write-heavy ingest workloads
- Ensure the MDS has sufficient memory to cache the fragment map

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_cache_memory_limit 8589934592  # 8 GiB
```

## Summary

CephFS directory fragmentation enables scalable handling of very large directories by splitting them into multiple RADOS objects using consistent hashing. Fragmentation is transparent to clients and helps avoid metadata bottlenecks in scenarios with directories containing hundreds of thousands of entries. In Rook-Ceph deployments, tuning `mds_bal_split_size` and pre-fragmenting large directories before populating them ensures consistent performance even at extreme directory scales.
