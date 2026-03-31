# How to Create and Manage Ceph Pools

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Pool, Storage, Management

Description: Learn how to create, configure, and manage Ceph storage pools for replicated and erasure-coded data using CLI commands and Rook CRDs.

---

## Pool Types

Ceph supports two pool types:

- **Replicated pools** - store multiple full copies of each object across OSDs
- **Erasure-coded pools** - split objects into data and coding chunks, using less storage at the cost of some flexibility

## Creating a Replicated Pool

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool create mypool 64 64 replicated
```

The parameters are: pool name, PG count, PGP count, and pool type. PG count and PGP count should be equal.

Set the replication factor:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set mypool size 3

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set mypool min_size 2
```

## Creating an Erasure-Coded Pool

First create an erasure code profile:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd erasure-code-profile set myprofile \
    k=4 m=2 crush-failure-domain=host

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool create ec-pool 64 64 erasure myprofile
```

This stores 4 data chunks and 2 parity chunks, tolerating 2 OSD failures while using 1.5x the raw storage (vs 3x for 3-replica).

## Listing and Inspecting Pools

```bash
# List all pools
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd lspools

# Detailed pool stats
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph df detail

# Pool configuration
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool ls detail
```

## Tagging Pools with Application Labels

Tag a pool with its intended application to enable Ceph to optimize behavior:

```bash
# For RBD (block storage)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool application enable mypool rbd

# For CephFS
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool application enable mypool cephfs

# For RGW (object storage)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool application enable mypool rgw
```

## Managing Pools via Rook CRDs

In Rook, define replicated pools using `CephBlockPool`:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: mypool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
    requireSafeReplicaSize: true
  parameters:
    pg_num: "64"
    pg_autoscale_mode: "on"
```

## Deleting a Pool

Deleting a pool is irreversible. You must explicitly allow it:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mon mon_allow_pool_delete true

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool delete mypool mypool --yes-i-really-really-mean-it

# Re-disable pool deletion
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mon mon_allow_pool_delete false
```

## Summary

Ceph pools are the primary storage namespace for your data. Create replicated pools for maximum availability and erasure-coded pools for storage efficiency. Always tag pools with the correct application label, and use `pg_autoscale_mode` to let Ceph manage PG counts automatically. In Rook, manage pools via `CephBlockPool`, `CephFilesystem`, and `CephObjectStore` CRDs.
