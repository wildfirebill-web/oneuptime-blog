# How to Understand ceph_health_status Metric

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Prometheus, Metric, Monitoring, Health, Kubernetes

Description: Interpret the ceph_health_status Prometheus metric to monitor Ceph cluster health states and build effective alerting rules for Rook-managed clusters.

---

## Overview

The `ceph_health_status` metric is the top-level indicator of Ceph cluster health in Prometheus. It maps Ceph's three health states to numeric values, making it easy to create alerts and dashboards for cluster status monitoring.

## Understanding the Metric Values

The `ceph_health_status` metric uses these values:

| Value | Ceph Status | Meaning |
|-------|-------------|---------|
| 0 | HEALTH_OK | All checks pass, cluster is healthy |
| 1 | HEALTH_WARN | Non-critical issues requiring attention |
| 2 | HEALTH_ERR | Critical issues requiring immediate action |

Query the current value:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph health
```

In Prometheus:

```promql
ceph_health_status
```

## Setting Up Prometheus Scraping

Ensure Rook's Prometheus metrics endpoint is accessible:

```yaml
spec:
  monitoring:
    enabled: true
    rulesNamespaceOverride: monitoring
```

Verify metrics are being exported:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  curl -s http://rook-ceph-mgr.rook-ceph.svc.cluster.local:9283/metrics | \
  grep ceph_health_status
```

Expected output:

```
# HELP ceph_health_status Health status of the cluster
# TYPE ceph_health_status gauge
ceph_health_status 0
```

## Querying Health Status in Prometheus

View current health in PromQL:

```promql
# Current cluster health
ceph_health_status

# Check if any cluster is not healthy
ceph_health_status > 0

# Check for errors only
ceph_health_status == 2
```

## Creating Alert Rules

Alert on health degradation:

```yaml
groups:
- name: ceph-health
  rules:
  - alert: CephHealthWarning
    expr: ceph_health_status == 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Ceph cluster is in HEALTH_WARN state"
      description: "Check ceph health detail for specific issues"

  - alert: CephHealthError
    expr: ceph_health_status == 2
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Ceph cluster is in HEALTH_ERR state"
      description: "Immediate investigation required"
```

Apply to the monitoring namespace:

```bash
kubectl apply -f ceph-health-alerts.yaml
```

## Building Grafana Panels

Create a status panel using `ceph_health_status`:

```javascript
// Grafana panel query
ceph_health_status

// Panel type: Stat
// Value mappings:
// 0 -> "OK" (green)
// 1 -> "WARN" (yellow)
// 2 -> "ERROR" (red)
```

## Correlating with Health Detail Metrics

Combine `ceph_health_status` with detail metrics for richer context:

```promql
# Show health status alongside specific check
ceph_health_status or ceph_health_detail
```

```bash
# Get health detail from Ceph directly
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph health detail
```

## Automating Responses

Use Alertmanager webhooks to trigger automated remediation when `ceph_health_status == 2`:

```yaml
receivers:
- name: ceph-oncall
  webhook_configs:
  - url: http://remediation-service/ceph-alert
    send_resolved: true
```

## Summary

The `ceph_health_status` metric provides a simple 0/1/2 numeric representation of Ceph cluster health (OK/WARN/ERR). Enable Rook's monitoring integration to expose this metric, create Prometheus alerts on values greater than 0, and map values to colored panels in Grafana for at-a-glance cluster health visibility.
