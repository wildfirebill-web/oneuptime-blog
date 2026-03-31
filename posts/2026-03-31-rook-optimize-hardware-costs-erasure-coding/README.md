# How to Optimize Ceph Hardware Costs with Erasure Coding

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Erasure Coding, Cost Optimization, Storage, Hardware

Description: Learn how erasure coding in Ceph reduces raw storage overhead compared to replication, lowering hardware costs while maintaining fault tolerance.

---

## Replication vs. Erasure Coding Cost Comparison

Three-way replication stores 3 copies of every byte, meaning you need 3x the raw capacity for 1x usable. Erasure coding can achieve the same fault tolerance with significantly less overhead.

A common EC profile like `k=4, m=2` (4 data chunks, 2 parity chunks) has an overhead factor of 1.5x instead of 3x:

```bash
# 3x replication: 100 TB usable requires 300 TB raw
# EC 4+2: 100 TB usable requires 150 TB raw
# Savings: 150 TB fewer drives
```

## Creating an Erasure-Coded Pool

```bash
# Create EC profile
ceph osd erasure-code-profile set ec42 \
  k=4 m=2 \
  crush-failure-domain=host

# Create EC pool
ceph osd pool create ec-data 64 64 erasure ec42

# Verify overhead
ceph osd erasure-code-profile get ec42
```

## Configuring EC Pools in Rook

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ec-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  erasureCoded:
    dataChunks: 4
    codingChunks: 2
  parameters:
    compression_mode: aggressive
```

## Calculating Savings Across Fleet Sizes

For a large deployment, the hardware savings compound:

```bash
# Scenario: 1 PB usable storage needed
# 3x replication: 3 PB raw = ~250 x 12 TB drives = ~$500,000
# EC 4+2:         1.5 PB raw = ~125 x 12 TB drives = ~$250,000
# Savings: ~$250,000 in drives alone
```

## Trade-offs to Consider

EC pools have higher CPU overhead for encode/decode operations. Benchmark before committing:

```bash
# Test EC pool write throughput
rados bench -p ec-data 60 write --no-cleanup

# Test replicated pool write throughput
rados bench -p replicated-data 60 write --no-cleanup

# Compare results - EC typically 10-30% slower on writes
```

## When EC Makes Financial Sense

EC is most cost-effective for:
- Cold or warm data that is read infrequently
- Object storage (RGW) workloads
- Backup and archive pools
- Large sequential workloads

Stick with replication for:
- Database volumes requiring low latency random I/O
- Frequently updated small files

## Choosing the Right EC Profile

```bash
# List available profiles and their overhead
ceph osd erasure-code-profile ls

# Common profiles and their storage efficiency:
# k=2,m=1: 1.5x overhead, tolerates 1 failure
# k=4,m=2: 1.5x overhead, tolerates 2 failures
# k=6,m=3: 1.5x overhead, tolerates 3 failures
# k=8,m=3: 1.375x overhead, tolerates 3 failures
```

## Summary

Erasure coding in Ceph can cut raw storage requirements by 30-50% compared to 3x replication, translating directly to hardware cost savings. For large deployments handling primarily sequential or infrequently-accessed data, EC profiles like `k=4,m=2` provide the same fault tolerance as 3-way replication at half the hardware cost.
