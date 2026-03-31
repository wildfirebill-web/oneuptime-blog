# How to Compare Replication vs Erasure Coding in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Erasure Coding, Replication, Storage Efficiency

Description: Compare Ceph replication and erasure coding across performance, storage efficiency, fault tolerance, and use case fit to make the right pool choice.

---

## Overview

Ceph supports two data protection strategies for pools: **replication** and **erasure coding**. Each has distinct trade-offs in storage efficiency, performance, and operational complexity. Choosing the right strategy depends on your workload, hardware, and tolerance for write latency.

## Replication

Replication stores N complete copies of every object across N different OSDs. With `size=3`, every byte of data uses 3x the raw storage.

```bash
# Create a 3-replica pool
ceph osd pool create replicated_pool 128 128 replicated
ceph osd pool set replicated_pool size 3
ceph osd pool set replicated_pool min_size 2
```

### Advantages of Replication

- Simple reads and writes (no encoding/decoding overhead)
- Low read latency (can read from any replica)
- Full support for all Ceph features (RBD, CephFS, RGW, RADOS)
- Easy recovery - simply copy the object to replace a failed replica

### Disadvantages of Replication

- High storage overhead - 3x raw capacity for 3-replica
- Not cost-effective for large cold datasets

## Erasure Coding

Erasure coding breaks each object into K data chunks and M parity chunks. The object can be reconstructed from any K of the K+M chunks.

```bash
# Create a 4+2 erasure coding profile (6 OSDs, 1.5x overhead)
ceph osd erasure-code-profile set my-ec-profile k=4 m=2 plugin=jerasure technique=reed_sol_van
ceph osd pool create ec_pool 128 128 erasure my-ec-profile
```

### Advantages of Erasure Coding

- Much better storage efficiency (e.g., 4+2 = 1.5x overhead vs 3x for replication)
- Higher fault tolerance per unit of storage
- Ideal for large object storage (RGW workloads)

### Disadvantages of Erasure Coding

- Higher write latency (encode, write K+M chunks)
- Read-modify-write cycles for partial updates (no true in-place updates for small writes)
- Limited RADOS object features (no omap support)
- More complex recovery (requires K chunks to reconstruct)
- Not directly usable as CephFS data pool without a replicated metadata pool

## Storage Efficiency Comparison

```text
Strategy          | Raw Overhead | Fault Tolerance
3-way replication | 3.0x         | Lose 2 OSDs
2-way replication | 2.0x         | Lose 1 OSD
EC 4+2            | 1.5x         | Lose 2 OSDs
EC 8+3            | 1.375x       | Lose 3 OSDs
EC 8+4            | 1.5x         | Lose 4 OSDs
EC 6+2            | 1.33x        | Lose 2 OSDs
```

## Performance Comparison

```text
Operation         | Replication        | Erasure Coding
Sequential Read   | Excellent          | Good (reads all K chunks)
Sequential Write  | Good               | Moderate (encode + K+M writes)
Random Read       | Excellent          | Good
Random Write      | Good               | Poor (read-modify-write cycles)
Partial Update    | Fast               | Slow (requires RMW)
Latency           | Low                | Higher (encoding overhead)
CPU Usage         | Low                | Higher (encoding/decoding)
```

## Choosing Based on Workload

Use **replication** for:
- RBD block storage (VMs, databases)
- CephFS metadata pools
- Workloads with small random writes
- Latency-sensitive applications

Use **erasure coding** for:
- Object storage (S3/RGW) with large objects
- Cold data archives and backups
- Environments where storage cost is primary concern
- Sequential read-heavy workloads

## Hybrid Approach

A common pattern is to use both - replication for metadata and active data, erasure coding for cold/bulk object storage:

```bash
# Create replicated metadata pool
ceph osd pool create cephfs_metadata 64 64 replicated

# Create EC data pool for bulk storage
ceph osd erasure-code-profile set ec-bulk k=6 m=2 plugin=jerasure
ceph osd pool create cephfs_data 128 128 erasure ec-bulk

# Create the CephFS filesystem
ceph fs new myfs cephfs_metadata cephfs_data
```

## Summary

Replication is the simpler and higher-performance choice for most workloads, at the cost of 2-3x storage overhead. Erasure coding significantly reduces storage overhead (as low as 1.33x) but introduces higher write latency and CPU overhead, making it best suited for large-object, sequential, read-heavy workloads like object storage. Many deployments use both: replication for active/metadata tiers and erasure coding for bulk object data.
