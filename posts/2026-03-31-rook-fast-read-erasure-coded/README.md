# How to Enable Fast Read for Erasure Coded Pools in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Erasure Coding, Pool, Performance

Description: Enable the fast_read optimization for Ceph erasure coded pools to improve read performance by fetching from all shards simultaneously.

---

Erasure coded (EC) pools in Ceph offer superior storage efficiency compared to replication, but reads are more complex - the cluster must reconstruct data from distributed shards. The `fast_read` flag enables a sub-read optimization that improves read latency by racing reads across available shards.

## How fast_read Works

In a standard EC pool read, Ceph queries the minimum required shards (`k` shards) and reconstructs the object. With `fast_read` enabled, Ceph sends requests to all available shards simultaneously and uses whichever `k` shards respond first. This reduces read latency when some OSDs are slow or under load, at the cost of slightly higher I/O amplification.

## When to Use fast_read

Use `fast_read` when:
- Your workload is read-heavy
- You have heterogeneous OSDs with varying response times
- You need consistent low-latency reads under partial OSD load

Avoid `fast_read` when:
- You are I/O constrained at the OSD level (it increases total I/O)
- Your cluster is CPU-bound (extra decoding overhead)
- You use HDDs where the extra seeks negate latency benefits

## Enable fast_read on an Erasure Coded Pool

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool set ec-pool fast_read true
```

Verify the setting:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool get ec-pool fast_read
```

## Create an EC Pool with fast_read via Rook

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
    fast_read: "1"
```

Note that `fast_read` is set as a string `"1"` (true) or `"0"` (false) in the parameters field.

## Verify the EC Pool Profile

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- bash -c "
  # List erasure code profiles
  ceph osd erasure-code-profile ls

  # Get profile details
  ceph osd erasure-code-profile get k4m2
"
```

Sample output:

```text
crush-failure-domain=host
k=4
m=2
plugin=jerasure
technique=reed_sol_van
```

## Disable fast_read

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool set ec-pool fast_read false
```

## Benchmark EC Pool Read Performance

Use the `rados bench` tool to measure read latency before and after enabling `fast_read`:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- bash -c "
  # Write test data
  rados -p ec-pool bench 60 write --no-cleanup

  # Sequential read benchmark
  rados -p ec-pool bench 60 seq

  # Random read benchmark
  rados -p ec-pool bench 60 rand
"
```

Compare latency (in ms) with and without `fast_read` to quantify the benefit for your workload.

## Summary

The `fast_read` flag in Ceph erasure coded pools improves read latency by racing requests to all available shards and using the fastest `k` responses. Enable it with `ceph osd pool set <pool> fast_read true` or via the Rook `CephBlockPool` `parameters` field. It is most beneficial for read-heavy workloads on NVMe or SSD-backed OSDs with heterogeneous response times.
