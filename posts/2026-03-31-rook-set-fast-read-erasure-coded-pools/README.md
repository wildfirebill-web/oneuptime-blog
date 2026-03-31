# How to Set fast_read for Erasure Coded Pools

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Erasure Coding, Performance, Pool

Description: Learn how to configure the fast_read option for Ceph erasure coded pools to reduce read latency by fetching data from all OSDs in parallel and using the fastest response.

---

## What fast_read Does

For erasure coded pools, Ceph normally reads only the minimum number of chunks required to reconstruct data (the k data chunks). The `fast_read` option changes this behavior: Ceph sends read requests to ALL shards simultaneously and uses whichever chunks arrive first.

This approach reduces tail latency - the worst-case read time caused by a single slow OSD - at the cost of additional network and IOPS overhead since more OSDs are queried per read.

## When to Use fast_read

Enable `fast_read` when:
- Read latency matters more than IOPS efficiency
- You have heterogeneous OSD speeds (mixing SSDs and HDDs)
- Network bandwidth is not a constraint
- You are using erasure coding for a read-heavy workload

Avoid `fast_read` when:
- IOPS efficiency matters (fast_read increases total OSD operations)
- Network bandwidth is limited
- Your cluster has consistent OSD performance

## Check Current fast_read Setting

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  ceph osd pool get my-ec-pool fast_read
"
```

Returns `0` (disabled) or `1` (enabled).

## Enable fast_read via Rook CRD

Set `fast_read` in the CephBlockPool parameters:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ec-pool
  namespace: rook-ceph
spec:
  failureDomain: osd
  erasureCoded:
    dataChunks: 4
    codingChunks: 2
  parameters:
    fast_read: "1"
```

Or for an object store data pool:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  dataPool:
    erasureCoded:
      dataChunks: 4
      codingChunks: 2
    parameters:
      fast_read: "1"
```

## Enable via CLI for Existing Pools

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  # Enable fast_read
  ceph osd pool set my-ec-pool fast_read 1

  # Disable fast_read
  ceph osd pool set my-ec-pool fast_read 0

  # Verify
  ceph osd pool get my-ec-pool fast_read
"
```

## Benchmark the Difference

Test read latency with and without fast_read using rados bench:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  # First, write test data
  rados bench -p my-ec-pool 30 write --no-cleanup

  # Benchmark with fast_read disabled
  ceph osd pool set my-ec-pool fast_read 0
  rados bench -p my-ec-pool 30 rand 2>&1 | grep -E 'Average|Stddev|Bandwidth'

  # Benchmark with fast_read enabled
  ceph osd pool set my-ec-pool fast_read 1
  rados bench -p my-ec-pool 30 rand 2>&1 | grep -E 'Average|Stddev|Bandwidth'

  # Cleanup
  rados -p my-ec-pool cleanup
"
```

Compare the `Average Latency` and `Stddev Latency` values. fast_read typically reduces both values when the cluster has OSD speed variance.

## Monitor IOPS Impact

Because fast_read sends requests to more OSDs per read, total OSD IOPS will increase:

```bash
# PromQL: Compare OSD read ops before/after enabling fast_read
sum(rate(ceph_osd_op_r[5m]))
```

If total IOPS increases significantly without a proportional throughput increase, consider whether the trade-off is worthwhile for your workload.

## Summary

The `fast_read` option for Ceph erasure coded pools reduces read latency by querying all data and parity shards simultaneously and using whichever chunks arrive first. Enable it via the `fast_read: "1"` parameter in Rook CephBlockPool or CephObjectStore CRDs when read latency is a priority and bandwidth overhead is acceptable. Benchmark with rados bench to confirm latency improvement before deploying to production.
