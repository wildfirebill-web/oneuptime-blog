# How to Use the ceph osd pool create Command

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CLI, Pool, OSD, Operations, Storage

Description: Create and configure Ceph storage pools using the ceph osd pool create command with correct PG counts, replication, and erasure coding settings.

---

## Introduction

Ceph stores data in pools, which are logical partitions of OSD space. Creating pools correctly - with appropriate PG counts, replication factors, and CRUSH rules - is fundamental to cluster performance and reliability. This guide covers the `ceph osd pool create` command and its key options.

## Basic Pool Creation

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

Create a replicated pool:

```bash
ceph osd pool create mypool 128
```

The number `128` is the PG (placement group) count. As a general rule: use `(OSDs * 100) / replicas` and round to nearest power of 2.

## Specifying Replication Factor

```bash
# Create pool with 3 replicas
ceph osd pool create mypool 128 128 replicated

# Set size (total copies) and min_size (minimum for writes)
ceph osd pool set mypool size 3
ceph osd pool set mypool min_size 2
```

## Creating an Erasure Coded Pool

Erasure coding provides storage efficiency over replication:

```bash
# Create an erasure code profile (4 data + 2 parity = 6 OSDs minimum)
ceph osd erasure-code-profile set my-profile \
  k=4 m=2 \
  crush-failure-domain=host

# Create the pool using this profile
ceph osd pool create ec-pool 64 64 erasure my-profile
```

EC pools provide 1.5x overhead vs 3x for 3-replica pools with similar durability.

## Setting Application Tags

Tell Ceph what application will use the pool:

```bash
# For RBD block storage
ceph osd pool application enable mypool rbd

# For CephFS
ceph osd pool application enable mypool cephfs

# For RGW object storage
ceph osd pool application enable mypool rgw
```

## Pool Quotas

Limit pool size to prevent runaway usage:

```bash
ceph osd pool set-quota mypool max_bytes 107374182400  # 100GB
ceph osd pool set-quota mypool max_objects 1000000
```

Check current quota:

```bash
ceph osd pool get-quota mypool
```

## Setting Compression

Enable compression on a pool for cold data:

```bash
ceph osd pool set mypool compression_mode aggressive
ceph osd pool set mypool compression_algorithm snappy
```

## Renaming and Deleting Pools

```bash
# Rename a pool
ceph osd pool rename oldname newname

# Delete a pool (requires confirmation flags)
ceph tell mon.\* injectargs '--mon-allow-pool-delete=true'
ceph osd pool delete mypool mypool --yes-i-really-really-mean-it
```

## Listing All Pools

```bash
ceph osd pool ls detail
ceph osd lspools
```

## Verifying Pool Creation

```bash
ceph osd pool stats mypool
ceph df | grep mypool
```

## Summary

The `ceph osd pool create` command is the building block for all Ceph storage. Choosing the right PG count (based on OSD count and replica factor), setting application tags, and optionally configuring erasure coding or compression determines both performance and storage efficiency. In Rook-managed clusters, pools are typically created via CephBlockPool or CephFilesystem CRDs, but understanding the CLI commands aids in troubleshooting and manual operations.
