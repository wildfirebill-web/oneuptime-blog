# How to Set pg_autoscale_bias Per Pool in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, PG Autoscaler, Pool, Performance

Description: Learn how to configure pg_autoscale_bias per Ceph pool to give high-priority pools more placement groups than their data size would suggest, improving performance for critical workloads.

---

## What pg_autoscale_bias Does

The `pg_autoscale_bias` parameter is a multiplier applied to the PG autoscaler's calculation for a specific pool. A bias of `1.0` is neutral (default). A bias of `2.0` causes the autoscaler to allocate twice as many PGs as it would based on data size alone.

Higher PG counts for a pool improve:
- Parallelism during recovery for that pool
- Data distribution granularity (reduced imbalance)
- Throughput for high-IOPS workloads

Lower PG counts (bias < 1.0) reduce:
- Memory overhead for less-critical pools
- PG management overhead

## View Current Autoscale Status

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  ceph osd pool autoscale-status
"
```

The `BIAS` column shows the current bias per pool. Default is `1.0`.

## Set pg_autoscale_bias via Rook CRD

Configure bias in the CephBlockPool parameters:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: high-priority-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
  parameters:
    pg_autoscale_mode: "on"
    pg_autoscale_bias: "2.0"    # 2x more PGs than data size suggests
```

For a lower-priority archival pool:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: archive-pool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
  parameters:
    pg_autoscale_mode: "on"
    pg_autoscale_bias: "0.5"   # Half as many PGs - saves memory
```

## Set via CLI

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  # Double PG allocation for database pool
  ceph osd pool set database-pool pg_autoscale_bias 2.0

  # Halve PG allocation for cold storage pool
  ceph osd pool set cold-storage pg_autoscale_bias 0.5

  # Reset to neutral
  ceph osd pool set backup-pool pg_autoscale_bias 1.0

  # Verify
  ceph osd pool autoscale-status
"
```

## Practical Bias Values by Use Case

| Use Case | Recommended Bias | Reason |
|----------|-----------------|--------|
| Production database (RBD) | 2.0 | High IOPS, recovery speed critical |
| Application block storage | 1.5 | Moderate priority |
| CephFS data pool | 1.0 | Default - balanced |
| Object storage (RGW) | 1.0 | Sequential, less sensitive |
| Cold archival storage | 0.5 | Low priority, save memory |
| Development/test pools | 0.5 | Minimize resource overhead |

## Monitor PG Count Changes

After setting bias, watch the autoscaler adjust PG counts:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  # View before and after states
  watch -n 10 'ceph osd pool autoscale-status | head -20'
"
```

The autoscaler adjusts PG counts gradually (splitting or merging PGs) to avoid impacting cluster performance.

## Verify PG Allocation After Bias Change

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  ceph osd pool ls detail | grep -E 'pool|pg_num'
"
```

Confirm that the high-priority pool now has proportionally more PGs than other same-sized pools.

## Total PG Budget Awareness

The total number of PGs across all pools should stay within Ceph's recommended range (50-200 PGs per OSD). If biases cause the total to exceed this, reduce biases or consider adding OSDs:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  # Check total PGs per OSD
  ceph pg stat
  ceph osd stat
"
```

## Summary

The `pg_autoscale_bias` parameter allows fine-grained control over how many PGs the autoscaler allocates to each pool relative to its data size. High-priority pools benefit from bias values above 1.0 for better parallelism and recovery speed, while archival or low-priority pools can use values below 1.0 to minimize memory overhead. Configure bias through Rook CRD parameters or the Ceph CLI, and monitor total PGs per OSD to stay within recommended bounds.
