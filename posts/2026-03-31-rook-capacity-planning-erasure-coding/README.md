# How to Plan Capacity with Erasure Coding

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Erasure Coding, Capacity, Planning, Storage Efficiency

Description: Plan Ceph storage capacity with erasure coding profiles, comparing storage efficiency, OSD requirements, performance trade-offs, and minimum node counts for different k+m configurations.

---

## Overview

Erasure coding stores data as mathematical chunks distributed across OSDs, providing configurable durability with better storage efficiency than replication. For large-scale object storage, erasure coding can double usable capacity compared to 3x replication.

## Erasure Coding Concepts

An EC profile `k+m` means:
- `k` = number of data chunks
- `m` = number of parity chunks
- Can tolerate loss of any `m` OSDs simultaneously
- Overhead factor = `(k+m)/k`
- Requires at least `k+m` OSDs

## Step 1 - Compare EC Profiles

```bash
#!/bin/bash
echo "=== Erasure Coding Efficiency Comparison ==="
echo "Profile | Overhead | Usable from 300TB | Nodes needed | Tolerates"
echo "--------|----------|-------------------|--------------|----------"

profiles=(
  "2+1:3:2:6"
  "4+2:6:4:6"
  "6+2:8:6:8"
  "6+3:9:6:9"
  "8+3:11:8:11"
  "8+4:12:8:12"
)

for p in "${profiles[@]}"; do
  IFS=: read PROFILE TOTAL K NODES <<< "$p"
  OVERHEAD=$(echo "scale=2; $TOTAL/$K" | bc)
  USABLE=$(echo "scale=1; 300 * $K / $TOTAL * 0.80" | bc)
  echo "$PROFILE | ${OVERHEAD}x | ${USABLE}TB | ${NODES} | $((TOTAL-K)) OSDs"
done
```

## Step 2 - Choose the Right EC Profile

```bash
# Light workloads / 3 nodes minimum: EC 2+1
# Matches 3-replica in fault tolerance, but 50% storage efficiency vs 33%

# Production object storage (6+ nodes): EC 4+2
# 66% efficiency, tolerates 2 simultaneous failures

# Large clusters (8+ nodes): EC 6+2
# 75% efficiency, best for cold storage / backups

# Check available OSDs before choosing
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd stat
```

## Step 3 - Create an EC Pool in Rook

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ec-pool
  namespace: rook-ceph
spec:
  erasureCoded:
    dataChunks: 4
    codingChunks: 2
  parameters:
    compression_mode: "none"
    bulk: "true"
```

Or create the EC profile and pool manually:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- bash

# Create EC profile
ceph osd erasure-code-profile set myprofile \
  k=4 m=2 \
  plugin=jerasure \
  technique=reed_sol_van

# Create pool with the profile
ceph osd pool create ecpool 128 erasure myprofile
ceph osd pool application enable ecpool rgw
```

## Step 4 - Capacity Calculations

```bash
#!/bin/bash
RAW_TB=600
K=4
M=2

USABLE=$(echo "scale=1; $RAW_TB * $K / ($K + $M) * 0.80" | bc)
OVERHEAD=$(echo "scale=2; ($K+$M)/$K" | bc)

echo "Raw capacity: ${RAW_TB} TB"
echo "EC profile: k=${K} m=${M}"
echo "Overhead factor: ${OVERHEAD}x"
echo "Usable capacity (80% threshold): ${USABLE} TB"
echo ""

# Compare to 3-replica
REPLICA_USABLE=$(echo "scale=1; $RAW_TB / 3 * 0.80" | bc)
SAVINGS=$(echo "scale=1; $USABLE - $REPLICA_USABLE" | bc)
echo "3-replica would give: ${REPLICA_USABLE} TB"
echo "EC savings: ${SAVINGS} TB additional usable"
```

## Step 5 - EC Performance Considerations

```bash
# EC read requires reading from k OSDs and computing
# EC write requires computing parity and writing to k+m OSDs
# Both operations add CPU overhead vs replication

# Benchmark EC pool performance
fio --name=ec-test \
  --ioengine=rados \
  --pool=ecpool \
  --rw=write \
  --bs=4m \
  --iodepth=16 \
  --numjobs=4 \
  --runtime=60

# EC pools have higher write latency - not suitable for databases
# Best for: object storage (S3/RGW), media, backup, archive
```

## Step 6 - EC for RGW (Object Storage)

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3   # Metadata always replicated
  dataPool:
    erasureCoded:
      dataChunks: 4
      codingChunks: 2
  preservePoolsOnDelete: true
```

## Step 7 - Monitor EC Pool Health

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph pg dump pools 2>/dev/null | grep ec

# Check incomplete PGs (EC-specific issue)
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph health detail | grep incomplete
```

## Summary

Erasure coding significantly improves storage efficiency compared to replication, with EC 4+2 giving ~66% storage efficiency vs 33% for 3-replica. The trade-offs are higher CPU usage, increased write latency, and minimum OSD count requirements. For object storage workloads where data is written once and read infrequently, erasure coding provides compelling economics at the cost of write performance. Plan EC pool deployments to have at least 1.5x the minimum OSD count for expansion headroom.
