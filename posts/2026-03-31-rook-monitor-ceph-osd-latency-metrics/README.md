# How to Monitor Ceph OSD Latency Metrics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitoring, OSD, Latency, Prometheus

Description: Learn how to monitor Ceph OSD latency metrics using Prometheus and Grafana to detect slow disks, identify bottlenecks, and alert on latency spikes before they impact workloads.

---

## Understanding OSD Latency

Ceph OSD latency has two primary components:

- **Apply latency**: Time for a write to be applied to the journal/WAL
- **Commit latency**: Time for a write to be committed to the backing store (BlueStore)

High latency in either metric can indicate disk problems, CPU saturation, or network issues between OSDs.

## Collect Latency Metrics via Prometheus

Rook exposes Ceph metrics through the embedded Prometheus exporter. Ensure it is enabled in your CephCluster:

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

Key Prometheus metrics for OSD latency:

```
ceph_osd_apply_latency_ms   - Apply latency per OSD in milliseconds
ceph_osd_commit_latency_ms  - Commit latency per OSD in milliseconds
```

## Query Latency in Prometheus

Find the top OSDs by apply latency:

```bash
# PromQL: Top 5 OSDs by apply latency
topk(5, ceph_osd_apply_latency_ms)

# Average apply latency across all OSDs
avg(ceph_osd_apply_latency_ms)

# P95 commit latency
quantile(0.95, ceph_osd_commit_latency_ms)
```

## Check Latency via CLI

For immediate triage, use the Ceph tools pod:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  ceph osd perf
"
```

This produces output like:

```
osd  commit_latency(ms)  apply_latency(ms)
  0                  2                  1
  1                 45                 40
  2                  3                  2
```

OSD 1 in this example shows unusually high latency and warrants investigation.

## Set Up Alerting Rules

Create a Prometheus alerting rule for high OSD latency:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ceph-osd-latency-alerts
  namespace: rook-ceph
spec:
  groups:
    - name: ceph-osd-latency
      rules:
        - alert: CephOSDHighApplyLatency
          expr: ceph_osd_apply_latency_ms > 50
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "OSD {{ $labels.ceph_daemon }} apply latency is {{ $value }}ms"
            description: "Apply latency above 50ms for 5 minutes indicates potential disk issues."
        - alert: CephOSDHighCommitLatency
          expr: ceph_osd_commit_latency_ms > 100
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "OSD {{ $labels.ceph_daemon }} commit latency is {{ $value }}ms"
```

## Grafana Dashboard Panels

Add these panels to your Ceph Grafana dashboard:

- Time series: `ceph_osd_apply_latency_ms` grouped by `ceph_daemon`
- Time series: `ceph_osd_commit_latency_ms` grouped by `ceph_daemon`
- Heatmap: latency distribution across all OSDs over time
- Stat panel: maximum latency across all OSDs (with threshold coloring)

## Diagnose a Slow OSD

If an OSD shows high latency, investigate the underlying disk:

```bash
# Check disk SMART health on the OSD node
smartctl -a /dev/sdb

# Check disk IO stats
iostat -x 1 5 /dev/sdb

# Check kernel dmesg for disk errors
dmesg | grep -E "sdb|error|I/O"
```

## Summary

Monitoring Ceph OSD latency with Prometheus involves enabling the Rook metrics exporter, querying `ceph_osd_apply_latency_ms` and `ceph_osd_commit_latency_ms`, setting up alerting rules for sustained high latency, and building Grafana dashboard panels to visualize trends. When latency spikes occur on a specific OSD, disk health checks using SMART and iostat help confirm whether a hardware replacement is needed.
