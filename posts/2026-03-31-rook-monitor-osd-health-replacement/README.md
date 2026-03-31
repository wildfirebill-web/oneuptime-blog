# How to Monitor OSD Health and Replacement Needs in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, OSD, Monitoring, Health, Kubernetes

Description: Learn how to proactively monitor OSD health in Rook-Ceph using SMART data, Ceph metrics, Prometheus alerts, and predictive failure indicators to plan timely replacements.

---

## Why Proactive OSD Monitoring Matters

Reactive disk replacement - replacing disks only after they fail - puts cluster redundancy at risk. Proactive monitoring using SMART data and Ceph performance metrics allows planned replacements before failures occur.

## Step 1: Monitor OSD Status with Ceph Commands

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd stat
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd df
```

Watch for OSDs that:
- Frequently go `down` and recover
- Have significantly higher latency than peers
- Show unusual utilization compared to identical disks

## Step 2: Check OSD Latency Trends

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd perf
```

Output includes `commit_latency_ms` and `apply_latency_ms` per OSD. Establish baselines and alert on deviation.

## Step 3: Enable SMART Monitoring via Node Exporter

Deploy the node-exporter with SMART monitoring enabled:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  template:
    spec:
      containers:
      - name: node-exporter
        args:
        - --collector.smartmon
        - --path.rootfs=/host
        volumeMounts:
        - name: dev
          mountPath: /dev
      volumes:
      - name: dev
        hostPath:
          path: /dev
```

Key SMART metrics to monitor:
- `smartmon_reallocated_sector_count` - sectors moved due to read errors
- `smartmon_uncorrectable_error_count` - permanent errors
- `smartmon_wear_leveling_count` - SSD wear indicator

## Step 4: Prometheus Alerts for OSD Health

```yaml
groups:
- name: ceph-osd-health
  rules:
  - alert: CephOSDDown
    expr: ceph_osd_up == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Ceph OSD {{ $labels.ceph_daemon }} is down"

  - alert: CephOSDHighLatency
    expr: ceph_osd_commit_latency_ms > 100
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "OSD {{ $labels.ceph_daemon }} commit latency > 100ms"

  - alert: CephOSDNearFull
    expr: ceph_osd_utilization > 85
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "OSD {{ $labels.ceph_daemon }} is {{ $value }}% full"
```

## Step 5: Identify Replacement Candidates

Run a weekly review script:

```bash
#!/bin/bash
echo "=== OSD Replacement Candidates ==="
echo "High latency OSDs:"
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd perf | awk 'NR>1 && $2 > 50 {print "OSD", $1, "commit_latency:", $2, "ms"}'

echo "Over-utilized OSDs:"
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd df | awk 'NR>1 && $7 > 80 {print "OSD", $1, "utilization:", $7, "%"}'
```

## Summary

Proactive OSD health monitoring in Rook combines Ceph's built-in metrics, SMART disk data, and Prometheus alerting to identify failing or degraded disks before they cause cluster health issues. Establishing latency baselines and scheduling regular reviews enables planned rather than emergency replacements.
