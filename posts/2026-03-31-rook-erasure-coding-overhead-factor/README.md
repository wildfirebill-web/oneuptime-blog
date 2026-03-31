# How to Calculate Erasure Coding Overhead Factor in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, ErasureCoding, Storage, Capacity

Description: Learn how to calculate the storage overhead factor for erasure coded pools in Ceph and compare it against replication to make informed capacity planning decisions.

---

Erasure coding provides storage efficiency by distributing redundancy mathematically instead of storing full copies. Understanding the overhead factor helps you plan cluster capacity and compare EC against replication for your specific fault tolerance needs.

## The Overhead Formula

For an erasure code profile with `k` data chunks and `m` coding (parity) chunks, the overhead factor is:

```text
overhead_factor = (k + m) / k
```

This means for every 1 GiB of usable data, the cluster stores `(k + m) / k` GiB of raw data.

## Common Profile Overhead Factors

```text
Profile (k=m)   Overhead Factor   Space Efficiency   Fault Tolerance
k=2, m=1        1.5x              66.7%              1 OSD failure
k=4, m=2        1.5x              66.7%              2 OSD failures
k=8, m=3        1.375x            72.7%              3 OSD failures
k=6, m=3        1.5x              66.7%              3 OSD failures
k=10, m=4       1.4x              71.4%              4 OSD failures

Replication (3x)  3.0x            33.3%              2 OSD failures
Replication (2x)  2.0x            50.0%              1 OSD failure
```

Note: k=4, m=2 achieves the same fault tolerance as 3-way replication but uses only 1.5x overhead instead of 3.0x - saving 50% raw storage.

## Calculating Raw Capacity Needed

If you need 100 TiB of usable EC storage with a k=4, m=2 profile:

```text
raw_needed = usable_data * overhead_factor
raw_needed = 100 TiB * 1.5 = 150 TiB raw
```

For the same usable capacity with 3-way replication:

```text
raw_needed = 100 TiB * 3.0 = 300 TiB raw
```

The EC pool saves 150 TiB of raw disk, a significant cost reduction for large-scale deployments.

## Checking Pool Space Usage in Ceph

```bash
ceph df detail
```

Look for the `USED` and `QUOTA BYTES` columns to see actual overhead vs usable capacity per pool.

For a specific EC pool:

```bash
ceph osd pool stats ec-pool
```

## Minimum OSD Count Requirement

For EC to be safe, you need at least `k + m` OSDs spread across failure domains:

```bash
# Verify enough hosts/OSDs exist for the profile
ceph osd erasure-code-profile get my-ec-profile
ceph osd tree | grep host | wc -l
```

If you have a k=4, m=2 profile, you need at least 6 OSD hosts (assuming host-level failure domain).

## Rook Capacity Planning Example

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ec-capacity-pool
  namespace: rook-ceph
spec:
  erasureCoded:
    dataChunks: 4    # k=4
    codingChunks: 2  # m=2, overhead = (4+2)/4 = 1.5x
```

With 10 nodes each having 10x 10 TiB drives (1000 TiB total raw), this pool provides:

```text
usable = 1000 TiB / 1.5 = ~666 TiB
```

vs 3-way replication:

```text
usable = 1000 TiB / 3.0 = ~333 TiB
```

## Summary

The EC overhead factor is `(k + m) / k`. Larger k values with fixed m reduce overhead and increase storage efficiency. A k=4, m=2 profile delivers the same 2-failure tolerance as 3-way replication at half the storage cost. Use this formula during capacity planning to determine raw disk requirements, minimum OSD counts, and cost savings compared to replicated pools.
