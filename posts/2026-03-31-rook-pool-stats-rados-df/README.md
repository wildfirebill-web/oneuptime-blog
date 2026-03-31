# How to View Pool Statistics with rados df and ceph osd pool stats

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Pool, Monitoring, RADOS

Description: Use rados df and ceph osd pool stats commands to view detailed pool-level storage statistics and I/O metrics in a Rook-Ceph cluster.

---

Monitoring pool-level statistics is essential for understanding storage consumption, I/O throughput, and object counts in a Ceph cluster. Two commands provide complementary views: `rados df` for storage usage and `ceph osd pool stats` for I/O operations.

## Access the Rook Toolbox

All Ceph CLI commands must be run inside the Rook toolbox pod:

```bash
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- bash
```

## Using rados df

`rados df` reports per-pool object counts and storage usage:

```bash
rados df
```

Sample output:

```text
POOL_NAME         USED  OBJECTS  CLONES  COPIES  MISSING_ON_PRIMARY  UNFOUND  DEGRADED  RD_OPS  RD   WR_OPS  WR   USED COMPR  UNDER COMPR
replicapool       4 GiB    1024       0    3072                   0        0         0   50231  2 GiB   12048  1 GiB        0 B          0 B
device_health_metrics 0 B    0       0       0                   0        0         0       0    0 B       0    0 B        0 B          0 B
```

Key columns:

| Column | Meaning |
|---|---|
| USED | Actual bytes stored in the pool |
| OBJECTS | Number of RADOS objects |
| COPIES | Total replicas (OBJECTS x replica count) |
| MISSING_ON_PRIMARY | Objects missing from primary OSD |
| DEGRADED | Under-replicated objects |
| RD_OPS / WR_OPS | Cumulative read/write operation counts |

## Filtering a Specific Pool

```bash
rados df --pool replicapool
```

Or use the shorter `-p` flag:

```bash
rados -p replicapool df
```

## Using ceph osd pool stats

`ceph osd pool stats` shows current I/O rates and recovery activity per pool:

```bash
ceph osd pool stats
```

Sample output:

```text
pool replicapool id 1
  nothing is going on

pool .rgw.root id 2
  client io 1.2 KiB/s rd, 512 B/s wr, 2 op/s rd, 1 op/s wr
```

For a specific pool:

```bash
ceph osd pool stats replicapool
```

## Monitor Ongoing Recovery

When a cluster is recovering from OSD failure or rebalancing, `ceph osd pool stats` shows recovery progress:

```text
pool replicapool id 1
  recovery io 256 MiB/s, 64 objects/s
  client io 44 MiB/s rd, 9 MiB/s wr, 11 op/s rd, 5 op/s wr
```

This is useful for estimating when recovery will complete.

## Combine with ceph df for Full Picture

```bash
# Overall cluster usage
ceph df

# Per-pool detailed breakdown
ceph df detail

# RADOS-level object statistics
rados df
```

## Script for Pool Health Summary

```bash
#!/bin/bash
echo "=== Pool Storage Usage ==="
rados df

echo ""
echo "=== Pool I/O Stats ==="
ceph osd pool stats

echo ""
echo "=== Degraded Objects ==="
ceph health detail | grep -i degraded
```

Save this as a daily check script to catch degraded objects or unusual I/O patterns early.

## Summary

`rados df` provides object counts and cumulative I/O statistics per pool, while `ceph osd pool stats` shows live I/O rates and recovery progress. Use both commands together when troubleshooting performance bottlenecks or verifying that data is evenly distributed across pools in your Rook-Ceph cluster.
