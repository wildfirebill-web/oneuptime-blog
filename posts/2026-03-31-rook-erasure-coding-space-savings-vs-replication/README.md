# How to Compare Erasure Coding Space Savings vs Replication in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Storage, Kubernetes, Erasure Coding

Description: Compare erasure coding space efficiency against replication in Ceph, with formulas, real-world examples, and guidance on when to use each approach.

---

## The Core Tradeoff

In Ceph, every pool uses either replication or erasure coding to protect against data loss. The choice affects storage efficiency, CPU usage, write latency, and operational complexity. Understanding the exact space overhead helps you make an informed decision before deploying production pools.

## Calculating Replication Overhead

With standard 3x replication, Ceph stores three full copies of every byte:

```text
Storage overhead = size * replicas
Overhead factor = replicas = 3x
Usable ratio = 1 / replicas = 33%
```

For 100 TB of raw disk, you get roughly 33 TB of usable storage. This is simple, predictable, and widely understood - but expensive.

For 2x replication:

```text
Usable ratio = 1 / 2 = 50%
```

## Calculating Erasure Coding Overhead

Erasure coding uses `k` data chunks and `m` parity chunks. The overhead formula is:

```text
Overhead factor = (k + m) / k
Usable ratio = k / (k + m)
```

Common configurations and their efficiency:

| Profile | k | m | Overhead Factor | Usable Ratio |
|---|---|---|---|---|
| k=2, m=1 | 2 | 1 | 1.5x | 67% |
| k=4, m=2 | 4 | 2 | 1.5x | 67% |
| k=6, m=2 | 6 | 2 | 1.33x | 75% |
| k=8, m=3 | 8 | 3 | 1.375x | 73% |
| k=8, m=4 | 8 | 4 | 1.5x | 67% |
| k=3, m=2 | 3 | 2 | 1.67x | 60% |

A `k=4,m=2` profile gives the same fault tolerance as 3x replication (both survive 2 simultaneous failures) but stores only 1.5x the data instead of 3x.

## Real World Comparison

Given 100 TB of raw disk:

```text
3x replication:   100 TB / 3 = 33.3 TB usable
k=4,m=2 EC:       100 TB / 1.5 = 66.7 TB usable
k=6,m=2 EC:       100 TB / 1.33 = 75.2 TB usable
```

For a 1 PB raw cluster:

```bash
# Calculate usable capacity
python3 -c "print(f'3x replication: {1000/3:.1f} TB')"
python3 -c "print(f'k=4,m=2 EC: {1000/1.5:.1f} TB')"
python3 -c "print(f'k=6,m=2 EC: {1000/1.33:.1f} TB')"
```

## Checking Actual Overhead in Ceph

To see the compression and erasure overhead on existing pools:

```bash
ceph df detail
```

The `STORED` column shows actual data, while `USED` shows raw disk used including overhead. You can calculate the effective ratio:

```bash
ceph osd pool stats my-ec-pool
```

## When Erasure Coding Saves the Most Space

EC is most beneficial when:

1. Raw disk cost dominates your budget
2. Data is write-once or sequentially written (backups, archives, media)
3. Objects are large (1 MB or more)
4. Read bandwidth matters more than read latency

Example: A video archive storing 500 TB of video files using `k=6,m=2` instead of 3x replication:

```text
3x replication needed:  500 TB * 3 = 1500 TB raw
k=6,m=2 EC needed:      500 TB * 1.33 = 665 TB raw
Disk savings:           835 TB
```

## When Replication Is Still Better

Despite the space savings, prefer replication when:

- Workloads require frequent partial writes (databases, RBD images)
- Low write latency is critical (EC requires writes across k+m OSDs)
- The pool is small (EC overhead per PG is higher with fewer OSDs)
- You need simple operations and fast recovery

```bash
# For a database workload, stay with replicated pool
ceph osd pool create db-pool replicated
ceph osd pool set db-pool size 3
```

## Summary

Erasure coding in Ceph can reduce storage overhead from 3x (replication) down to 1.33x-1.5x for typical profiles, doubling or tripling usable capacity from the same raw hardware. The `k=4,m=2` and `k=6,m=2` profiles offer the best balance of fault tolerance and efficiency. Use erasure coding for large-object sequential workloads and keep replicated pools for databases and latency-sensitive applications.
