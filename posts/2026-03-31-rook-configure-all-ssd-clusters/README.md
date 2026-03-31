# How to Configure Ceph for All-SSD Clusters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, SSD, Performance, Hardware, Configuration

Description: Configure Ceph and Rook optimally for all-SSD deployments by tuning BlueStore cache sizes, OSD counts, and recovery parameters for flash-specific performance.

---

## Why All-SSD Ceph Needs Different Tuning

SSD characteristics differ fundamentally from HDDs: lower latency, higher IOPS, no rotational penalty, and different failure modes. Default Ceph settings are optimized for HDDs and leave significant SSD performance on the table.

## Rook CephCluster Configuration for SSD

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  storage:
    useAllDevices: false
    nodes:
      - name: "worker-01"
        devices:
          - name: "sdb"
            config:
              deviceClass: ssd
          - name: "sdc"
            config:
              deviceClass: ssd
  resources:
    osd:
      requests:
        cpu: "2"
        memory: "4Gi"
      limits:
        cpu: "4"
        memory: "8Gi"
```

## Tune BlueStore for SSD

```bash
# Larger BlueStore cache for SSDs (default is 1 GB for HDDs)
ceph config set osd bluestore_cache_size_ssd 4294967296   # 4 GB

# Reduce minimum allocation size for better space efficiency on SSDs
ceph config set osd bluestore_min_alloc_size_ssd 4096     # 4 KB

# Disable SSD-specific rotational hints
ceph config set osd osd_class_update_on_start true
```

## Increase OSD Thread Counts

```bash
# SSDs can handle more concurrent operations
ceph config set osd osd_op_num_threads_per_shard 2
ceph config set osd osd_op_num_shards 8

# Increase client operations queue depth
ceph config set osd osd_max_ops 512
```

## Optimize Recovery for SSDs

```bash
# SSDs can handle much faster recovery than HDDs
ceph config set osd osd_recovery_max_active_ssd 5
ceph config set osd osd_max_backfills 4

# Reduce recovery sleep time (0 for SSDs, non-zero for HDDs)
ceph config set osd osd_recovery_sleep_ssd 0
ceph config set osd osd_backfill_scan_max 256
```

## Create SSD-Optimized CRUSH Rule

```bash
# CRUSH rule targeting SSD device class
ceph osd crush rule create-replicated ssd-rule default host ssd
```

## Create SSD Storage Pool

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ssd-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
  parameters:
    crush_rule: ssd-rule
    # Enable inline compression for SSDs with compressible workloads
    compression_mode: "passive"
    compression_algorithm: "lz4"
```

## Monitor SSD-Specific Metrics

```bash
# Check SSD IOPS and latency via OSD perf
ceph daemon osd.0 perf dump | python3 -c "
import json, sys
d = json.load(sys.stdin)
ops = d.get('osd', {})
print('Read lat (ms):', ops.get('op_r_latency', {}).get('avgcount', 0))
print('Write lat (ms):', ops.get('op_w_latency', {}).get('avgcount', 0))
"

# Monitor BlueStore SSD cache hit rate
ceph daemon osd.0 perf dump | python3 -c "
import json, sys
d = json.load(sys.stdin)
bs = d.get('bluestore', {})
hits = bs.get('bluestore_cache_hits', 0)
misses = bs.get('bluestore_cache_misses', 0)
if hits + misses > 0:
    print(f'Cache hit rate: {hits/(hits+misses)*100:.1f}%')
"
```

## Summary

All-SSD Ceph clusters benefit from larger BlueStore caches, reduced minimum allocation sizes, increased recovery parallelism, and SSD-specific CRUSH rules. These tunings allow SSDs to deliver their full IOPS potential while maintaining Ceph's reliability guarantees.
