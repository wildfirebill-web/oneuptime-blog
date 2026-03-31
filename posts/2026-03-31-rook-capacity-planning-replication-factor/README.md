# How to Plan Capacity with Replication Factor

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Replication, Capacity, Planning, Pool, Durability

Description: Plan Ceph storage capacity with replication factor by calculating the relationship between usable storage, raw capacity, fault tolerance, and performance trade-offs for 2x vs 3x replication.

---

## Overview

Replication factor in Ceph determines how many copies of each data object are stored. Higher replication provides better durability and read performance but reduces usable capacity. This guide covers choosing and planning for different replication factors.

## Replication Factor Options

| Factor | Copies | Usable % of Raw | Tolerates Failures |
|--------|--------|-----------------|-------------------|
| 2x | 2 | 50% | 1 OSD or node |
| 3x | 3 | 33% | 2 OSDs or nodes |
| 4x | 4 | 25% | 3 OSDs or nodes |

## Step 1 - Choose Replication Factor

```bash
# For most production workloads: 3x replication
# For non-critical data or cost-sensitive: 2x
# For compliance/archival requiring extreme durability: consider EC or 4x

# Check current pool replication
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd dump | grep -E "pool|size|min_size"
```

## Step 2 - Calculate Capacity Requirements

```bash
#!/bin/bash
USABLE_NEEDED_TB=$1  # First argument
REPLICATION=$2       # Second argument: 2 or 3

# Calculate raw needed at 80% utilization
RAW_NEEDED=$(echo "scale=1; $USABLE_NEEDED_TB * $REPLICATION / 0.80" | bc)

echo "Usable needed: ${USABLE_NEEDED_TB} TB"
echo "Replication factor: ${REPLICATION}x"
echo "Raw capacity needed: ${RAW_NEEDED} TB"

# bash capacity_calc.sh 100 3
# Usable needed: 100 TB
# Replication factor: 3x
# Raw capacity needed: 375.0 TB
```

## Step 3 - Configure Replication in Rook

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 3          # Total copies
    requireSafeReplicaSize: true  # Prevent writes if < min_size copies available
    replicasPerFailureDomain: 1
```

For 2-replica with weaker durability (test/dev):

```yaml
spec:
  replicated:
    size: 2
    requireSafeReplicaSize: false
```

## Step 4 - Mixed Replication Pools

Use different replication factors for different data tiers:

```yaml
# Hot tier: 3x for databases
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: hot-pool
spec:
  replicated:
    size: 3
  deviceClass: ssd

---
# Warm tier: 2x for less critical data
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: warm-pool
spec:
  replicated:
    size: 2
  deviceClass: hdd
```

## Step 5 - Minimum Replicas for Writes

The `min_size` setting controls write availability:

```bash
# Set minimum replicas required for writes
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool set replicapool min_size 2

# With size=3 and min_size=2:
# Writes succeed if 2 of 3 copies acknowledge
# Cluster degrades when < 3 copies but stays writable
```

## Step 6 - Replication and Recovery Capacity

During OSD failure and recovery, Ceph remaps placement groups. You need spare capacity:

```bash
# Rule: Keep 20-25% free for recovery overhead
# With 300 TB raw, 3x replication:
# Full usable = 100 TB theoretical
# Safe usable = 80 TB (80% threshold)
# Recovery buffer = 20 TB

# If you fill to 90%:
# During OSD failure, recovery may hit full threshold
# This causes I/O pausing - avoid exceeding 80%
```

## Step 7 - Monitor Replication Health

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph health detail | grep -E "replica|degraded|under"
```

Check per-pool replica status:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph pg stat | grep -E "degraded|peering|stale"
```

## Capacity Planning Summary

```bash
#!/bin/bash
echo "=== Ceph Replication Capacity Calculator ==="

for REPLICA in 2 3; do
  for RAW in 100 200 500 1000; do
    USABLE=$(echo "scale=1; $RAW / $REPLICA * 0.80" | bc)
    echo "${RAW}TB raw x${REPLICA} = ${USABLE}TB usable"
  done
  echo "---"
done
```

## Summary

Replication factor is the most significant lever in Ceph capacity planning. Three-way replication provides the best balance of durability, read performance, and operational simplicity for production workloads. Plan your raw capacity purchases by starting from your usable storage target, multiplying by the replication factor, then adding 25% for the safety utilization threshold. Never exceed 80% cluster utilization in production to ensure recovery operations complete without hitting hard limits.
