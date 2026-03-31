# How to Monitor BlueStore Performance Metrics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, BlueStore, Monitoring, Performance, Metrics, Prometheus, Grafana

Description: Monitor BlueStore OSD performance metrics using Ceph's perf counters, Prometheus, and Grafana to identify bottlenecks.

---

## Overview

BlueStore exposes a rich set of performance counters covering write/read paths, cache efficiency, deferred writes, and RocksDB compaction. Monitoring these metrics helps identify performance bottlenecks in individual OSDs and across the cluster. This guide covers collecting and interpreting BlueStore performance metrics.

## BlueStore Performance Counter Categories

BlueStore metrics are organized into categories:

| Category | What It Measures |
|---|---|
| `bluestore` | Direct BlueStore I/O operations |
| `rocksdb` | RocksDB metadata operations |
| `bluefs` | BlueFS filesystem operations |
| `osd` | OSD-level request handling |

## Reading Performance Counters via Admin Socket

```bash
# Dump all BlueStore counters for OSD 0
ceph daemon osd.0 perf dump | python3 -c "
import sys, json
data = json.load(sys.stdin)
print('=== BlueStore Metrics ===')
for k, v in data.get('bluestore', {}).items():
    print(f'{k}: {v}')
" | head -60
```

## Key BlueStore Metrics to Watch

### Write Metrics

```bash
ceph daemon osd.0 perf dump | python3 -c "
import sys, json
data = json.load(sys.stdin)
bs = data.get('bluestore', {})
metrics = [
    'bluestore_write_big',        # Large direct writes
    'bluestore_write_small',      # Small writes via WAL
    'bluestore_write_deferred',   # Deferred writes pending
    'bluestore_write_pad_bytes',  # Wasted padding bytes
]
for m in metrics:
    print(m, ':', bs.get(m, 'N/A'))
"
```

### Cache Metrics

```bash
ceph daemon osd.0 perf dump | python3 -c "
import sys, json
data = json.load(sys.stdin)
bs = data.get('bluestore', {})
metrics = [
    'bluestore_cache_hits',
    'bluestore_cache_misses',
    'bluestore_cache_bytes',
    'bluestore_cache_trim_max_skip_pinned',
]
for m in metrics:
    print(m, ':', bs.get(m, 'N/A'))
"
```

### RocksDB Metrics

```bash
ceph daemon osd.0 perf dump | python3 -c "
import sys, json
data = json.load(sys.stdin)
rdb = data.get('rocksdb', {})
for k, v in list(rdb.items())[:20]:
    print(k, ':', v)
"
```

## Prometheus Integration

Enable the Prometheus module in Ceph:

```bash
ceph mgr module enable prometheus
ceph mgr services | grep prometheus
```

Key BlueStore Prometheus metrics:

```promql
# OSD write latency (seconds)
rate(ceph_osd_op_w_latency_sum[5m]) / rate(ceph_osd_op_w_latency_count[5m])

# BlueStore cache hit ratio
ceph_bluestore_cache_hits / (ceph_bluestore_cache_hits + ceph_bluestore_cache_misses)

# OSD write bytes per second
rate(ceph_osd_op_w_in_bytes[5m])
```

## Grafana Dashboard for BlueStore

Import the official Ceph Grafana dashboards:

```bash
# Apply monitoring examples from Rook
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/deploy/examples/monitoring/service-monitor.yaml
```

Or manually import dashboard ID `2842` from Grafana.com which includes BlueStore panels.

## Collecting Metrics for All OSDs

```bash
#!/bin/bash
echo "OSD, WriteLatency_ms, CacheHitRatio, DeferredWrites"
for OSD in $(ceph osd ls | head -10); do
  DATA=$(ceph daemon osd.$OSD perf dump 2>/dev/null)
  if [ -n "$DATA" ]; then
    echo $DATA | python3 -c "
import sys, json
data = json.load(sys.stdin)
bs = data.get('bluestore', {})
osd = data.get('osd', {})

hits = bs.get('bluestore_cache_hits', {}).get('avgcount', 0)
misses = bs.get('bluestore_cache_misses', {}).get('avgcount', 0)
total = hits + misses
hit_ratio = hits / total if total > 0 else 0

write_lat = osd.get('op_w_latency', {}).get('sum', 0)
write_cnt = osd.get('op_w_latency', {}).get('avgcount', 1)
avg_lat_ms = (write_lat / write_cnt * 1000) if write_cnt > 0 else 0

deferred = bs.get('bluestore_write_deferred', {}).get('avgcount', 0)
print(f'$OSD, {avg_lat_ms:.2f}, {hit_ratio:.3f}, {deferred}')
"
  fi
done
```

## Summary

Monitoring BlueStore performance requires combining admin socket perf counters for per-OSD detail with Prometheus metrics for cluster-wide trends. Write latency histograms, cache hit ratios, deferred write volumes, and RocksDB compaction rates are the most critical indicators of BlueStore health. Grafana dashboards built on Ceph's Prometheus exporter provide the operational visibility needed to detect and diagnose performance regressions before they impact users.
