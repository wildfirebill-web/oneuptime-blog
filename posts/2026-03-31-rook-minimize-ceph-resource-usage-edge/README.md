# How to Minimize Ceph Resource Usage for Edge Deployments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Edge Computing, Resource Optimization, Performance

Description: Learn how to reduce Ceph CPU, memory, and disk usage for resource-constrained edge deployments while maintaining acceptable storage performance.

---

Edge nodes often have as little as 4-8 GB RAM and 4 CPU cores shared across all workloads. Minimizing Ceph's resource footprint lets you co-locate storage with application workloads without starving either.

## Memory Optimization

The biggest consumer is BlueStore cache. Reduce it aggressively:

```bash
# Target memory per OSD (default 4 GB, reduce to 1 GB)
ceph config set osd osd_memory_target 1073741824

# BlueStore cache limits
ceph config set osd bluestore_cache_size_hdd 536870912
ceph config set osd bluestore_cache_size_ssd 1073741824

# RocksDB block cache
ceph config set osd bluestore_rocksdb_options "max_write_buffer_number=2,min_write_buffer_number_to_merge=1,write_buffer_size=33554432"
```

## CPU Optimization

Reduce worker threads and background I/O:

```bash
# Reduce OSD worker threads
ceph config set osd osd_op_num_threads_per_shard 1
ceph config set osd osd_op_num_shards 4

# Limit recovery workers
ceph config set osd osd_max_backfills 1
ceph config set osd osd_recovery_max_active 1
```

## Rook Resource Limits

Set conservative resource limits in the CephCluster spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  resources:
    mon:
      requests:
        cpu: "100m"
        memory: "256Mi"
      limits:
        cpu: "250m"
        memory: "512Mi"
    osd:
      requests:
        cpu: "200m"
        memory: "512Mi"
      limits:
        cpu: "500m"
        memory: "1Gi"
    mgr:
      requests:
        cpu: "50m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "256Mi"
    crashcollector:
      requests:
        cpu: "15m"
        memory: "32Mi"
      limits:
        cpu: "50m"
        memory: "64Mi"
```

## Disabling Non-Essential Services

Turn off services not needed at the edge:

```yaml
spec:
  dashboard:
    enabled: false
  monitoring:
    enabled: false
  mgr:
    modules:
    - name: pg_autoscaler
      enabled: false
    - name: telemetry
      enabled: false
```

## Reducing the Number of Daemons

Use a single MGR on edge:

```yaml
spec:
  mgr:
    count: 1
```

## Lowering PG Count

Fewer PGs means less metadata memory:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: edge-pool
  namespace: rook-ceph
spec:
  replicated:
    size: 2
  parameters:
    pg_num: "16"
    pgp_num: "16"
```

## Monitoring Resource Usage

```bash
kubectl -n rook-ceph top pods
ceph tell osd.0 perf dump | python3 -m json.tool | grep -E "heap|memory"
```

## Summary

Minimizing Ceph resource usage on edge nodes requires tuning BlueStore memory targets, reducing worker thread counts, setting appropriate Kubernetes resource limits, disabling unused modules and services, and choosing small PG counts for the pool size. These changes allow Ceph to share hardware effectively with application workloads at the edge.
