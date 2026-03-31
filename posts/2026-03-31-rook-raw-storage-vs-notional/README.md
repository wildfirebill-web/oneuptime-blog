# How to Understand Raw Storage vs Notional Usage in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Storage, Capacity, Replication

Description: Understand the difference between raw storage consumption and notional (logical) data usage in Ceph to accurately plan capacity and avoid nearfull conditions.

---

One of the most common sources of confusion in Ceph capacity planning is the difference between how much data clients think they have stored (notional/logical) and how much raw disk space is actually consumed. Understanding this distinction is critical for avoiding unexpected `OSD_NEARFULL` conditions.

## The Raw vs Logical Gap

When you write 1 GB of data to a 3-replica Ceph pool:
- **Logical (notional) stored**: 1 GiB (what your application sees)
- **Raw used**: 3 GiB (actual bytes on disk across three OSDs)

This 3x multiplier applies to every byte of client data. For erasure coded pools, the ratio is lower (typically 1.3x-1.5x), but still greater than 1.

## View Both Values in ceph df

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph df
```

```text
--- POOLS ---
POOL               ID  STORED   USED     %USED  MAX AVAIL
replicapool         1  200 GiB  600 GiB  3.57   5 TiB
```

- **STORED**: 200 GiB - logical data written by clients
- **USED**: 600 GiB - actual raw bytes on OSDs (200 GiB x 3 replicas)
- **MAX AVAIL**: 5 TiB - maximum additional logical data this pool can accept

## Calculate the Overhead Multiplier

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph df --format json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for pool in data['pools']:
    stored = pool['stats']['stored']
    used = pool['stats']['bytes_used']
    if stored > 0:
        ratio = used / stored
        print(f'{pool[\"name\"]}: {ratio:.2f}x overhead ({stored//2**30} GiB logical -> {used//2**30} GiB raw)')
"
```

## Raw Capacity vs Usable Capacity

Raw capacity is the sum of all OSD disk sizes. Usable capacity is what you can actually store:

```text
Usable = Raw total / replication_factor

Example:
  3 nodes x 10 TiB = 30 TiB raw
  Replication factor = 3
  Usable = 30 / 3 = 10 TiB
```

For erasure coding (k=4, m=2):
```text
EC overhead = (k + m) / k = 6 / 4 = 1.5x
Usable = 30 / 1.5 = 20 TiB
```

## OSD Nearfull Thresholds Are Raw-Based

The `OSD_NEARFULL` warning triggers based on raw OSD utilization, not logical pool usage. A pool at 30% logical capacity can have OSDs at 90% raw capacity if data is not evenly distributed or if the replication factor is high.

Check per-OSD usage:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd df | sort -k7 -n -r | head -10
```

## Capacity Planning Formula

For a target maximum logical usage with safety headroom:

```python
# Variables
raw_total_tib = 60       # total raw capacity
replication = 3          # replica count
safety_factor = 0.75     # use only 75% of raw capacity

usable = (raw_total_tib / replication) * safety_factor
print(f"Safe usable capacity: {usable:.1f} TiB")
# Output: Safe usable capacity: 15.0 TiB
```

## Summary

In Ceph, raw storage consumption is always a multiple of logical stored data - 3x for 3-replica pools, 1.5x for k=4,m=2 EC pools. The `ceph df` `STORED` column shows logical data, while `USED` shows actual raw consumption. Plan capacity using `MAX AVAIL` (which already accounts for replication), and configure nearfull thresholds based on raw OSD utilization to avoid running out of space unexpectedly.
