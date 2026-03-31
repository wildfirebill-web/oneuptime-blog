# How to Monitor Ceph Throughput Metrics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitoring, Throughput, Prometheus, Grafana

Description: Learn how to monitor Ceph cluster throughput metrics using Prometheus to track read and write bandwidth, identify bottlenecks, and alert on abnormal traffic patterns.

---

## Key Throughput Metrics in Ceph

Ceph exposes cluster-level and per-OSD throughput metrics through Prometheus. The most important ones are:

- `ceph_cluster_total_bytes_read` - cumulative bytes read from the cluster
- `ceph_cluster_total_bytes_written` - cumulative bytes written to the cluster
- `ceph_osd_op_r_out_bytes` - bytes read out per OSD
- `ceph_osd_op_w_in_bytes` - bytes written in per OSD

Use `rate()` in PromQL to convert these counters to bytes-per-second throughput.

## Enable Prometheus Metrics

Verify that monitoring is enabled in your CephCluster resource:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  monitoring:
    enabled: true
    rulesNamespace: rook-ceph
```

After applying, check that the metrics endpoint is responding:

```bash
kubectl -n rook-ceph get servicemonitor
```

## Query Throughput in Prometheus

Calculate cluster read and write throughput in bytes per second:

```bash
# Cluster write throughput (bytes/sec)
rate(ceph_cluster_total_bytes_written[5m])

# Cluster read throughput (bytes/sec)
rate(ceph_cluster_total_bytes_read[5m])

# Per-OSD write throughput
rate(ceph_osd_op_w_in_bytes[5m])

# Top 5 OSDs by write throughput
topk(5, rate(ceph_osd_op_w_in_bytes[5m]))
```

## Check Throughput via CLI

For immediate visibility without Prometheus:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  ceph -s
  ceph osd pool stats
"
```

The `ceph -s` output shows current read/write throughput in the `io:` section:

```
  cluster:
    id:     ...
    health: HEALTH_OK

  services:
    io:
      client:   read: 150 MiB/s
                write: 80 MiB/s
```

## Benchmark Throughput with rados bench

Measure maximum throughput of a specific pool:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  # Sequential write benchmark
  rados bench -p rbd 60 write --no-cleanup

  # Sequential read benchmark
  rados bench -p rbd 60 seq

  # Random read benchmark
  rados bench -p rbd 60 rand
"
```

## Set Up Throughput Alerts

Alert when throughput drops below expected levels (indicating a bottleneck) or spikes unexpectedly:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ceph-throughput-alerts
  namespace: rook-ceph
spec:
  groups:
    - name: ceph-throughput
      rules:
        - alert: CephHighWriteThroughput
          expr: rate(ceph_cluster_total_bytes_written[5m]) > 500e6
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Ceph write throughput above 500 MB/s for 10 minutes"
        - alert: CephLowReadThroughput
          expr: rate(ceph_cluster_total_bytes_read[5m]) < 1e6
          for: 30m
          labels:
            severity: info
          annotations:
            summary: "Ceph read throughput below 1 MB/s - cluster may be underutilized"
```

## Grafana Dashboard Panels

Build a throughput dashboard with these panels:

- Time series: cluster read and write throughput (MB/s) over 24 hours
- Time series: per-OSD throughput to identify imbalanced nodes
- Stat panel: current read and write throughput with color thresholds
- Heatmap: per-OSD write throughput distribution

## Summary

Monitoring Ceph throughput requires querying Prometheus with `rate()` on cumulative counter metrics to derive bytes-per-second values for both reads and writes. Combining Prometheus alerts for throughput anomalies with Grafana dashboards gives operators visibility into normal traffic patterns and early warning of bottlenecks or unexpected workload spikes.
