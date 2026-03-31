# How to Automate Ceph Capacity Alerts and Notifications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Alerting, Capacity Planning, Prometheus

Description: Learn how to set up automated capacity alerts for Ceph using Prometheus alerting rules and notification channels to prevent storage exhaustion.

---

Running out of Ceph storage capacity is catastrophic - it causes all write I/O to fail cluster-wide. Automated capacity alerts give you advance warning to add storage before you hit critical thresholds.

## Understanding Ceph Capacity Thresholds

Ceph has built-in capacity warnings:
- **nearfull** ratio (default 85%) - HEALTH_WARN
- **backfillfull** ratio (default 90%) - prevents backfill
- **full** ratio (default 95%) - blocks all writes

Check current settings:

```bash
ceph osd dump | grep -E "full_ratio|nearfull|backfillfull"
```

## Prometheus Alerting Rules

Create comprehensive capacity alerts:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ceph-capacity-alerts
  namespace: rook-ceph
spec:
  groups:
  - name: ceph-capacity
    rules:
    - alert: CephClusterCapacity60Percent
      expr: |
        (ceph_cluster_total_used_raw_bytes / ceph_cluster_total_bytes) * 100 > 60
      for: 15m
      labels:
        severity: info
      annotations:
        summary: "Ceph cluster is {{ $value | humanize }}% full"
        description: "Plan capacity expansion within 30 days"

    - alert: CephClusterCapacity75Percent
      expr: |
        (ceph_cluster_total_used_raw_bytes / ceph_cluster_total_bytes) * 100 > 75
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Ceph cluster is {{ $value | humanize }}% full"
        description: "Add storage within 1 week"

    - alert: CephClusterCapacity85Percent
      expr: |
        (ceph_cluster_total_used_raw_bytes / ceph_cluster_total_bytes) * 100 > 85
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Ceph cluster is {{ $value | humanize }}% full - CRITICAL"
        description: "Immediate action required - cluster approaching full"

    - alert: CephPoolCapacity90Percent
      expr: |
        (ceph_pool_bytes_used / ceph_pool_max_bytes) * 100 > 90
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Ceph pool {{ $labels.name }} is {{ $value | humanize }}% full"
```

## Alertmanager Configuration

Route capacity alerts to appropriate teams:

```yaml
route:
  receiver: default
  routes:
  - match:
      alertname: CephClusterCapacity85Percent
    receiver: oncall-pagerduty
  - match:
      alertname: CephClusterCapacity75Percent
    receiver: storage-team-slack

receivers:
- name: storage-team-slack
  slack_configs:
  - api_url: https://hooks.slack.com/services/xxx/yyy/zzz
    channel: '#storage-ops'
    text: >
      Ceph Capacity Alert: {{ .CommonAnnotations.summary }}
      {{ .CommonAnnotations.description }}

- name: oncall-pagerduty
  pagerduty_configs:
  - service_key: <pagerduty-key>
    description: "{{ .CommonAnnotations.summary }}"
```

## Growth Rate Prediction Alert

Alert when the cluster will be full within 7 days:

```yaml
- alert: CephCapacityGrowthRateHigh
  expr: |
    predict_linear(
      ceph_cluster_total_used_raw_bytes[24h], 7 * 24 * 3600
    ) / ceph_cluster_total_bytes > 0.90
  for: 30m
  labels:
    severity: warning
  annotations:
    summary: "Ceph cluster will be 90% full in 7 days at current growth rate"
```

## Capacity Dashboard Queries

For Grafana:

```promql
# Current usage percentage
(ceph_cluster_total_used_raw_bytes / ceph_cluster_total_bytes) * 100

# Time until full (hours) at current rate
(ceph_cluster_total_bytes - ceph_cluster_total_used_raw_bytes) /
  deriv(ceph_cluster_total_used_raw_bytes[1h]) / 3600
```

## Summary

Automating Ceph capacity alerts requires a layered approach: early warnings at 60% and 75% full for planning, critical alerts at 85% for immediate action, and growth rate prediction to catch accelerating usage before it becomes critical. Routing different severity levels to different channels (Slack for warnings, PagerDuty for critical) ensures the right team responds with appropriate urgency.
