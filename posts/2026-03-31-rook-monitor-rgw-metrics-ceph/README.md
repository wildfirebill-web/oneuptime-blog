# How to Monitor RGW Metrics in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitoring, Metric, Prometheus, Grafana, Object Storage

Description: Monitor Ceph RGW performance and health metrics using Prometheus and Grafana to track request rates, latency, error rates, and storage utilization.

---

Monitoring Ceph RGW is essential for capacity planning, performance tuning, and incident detection. RGW exposes metrics via Prometheus and via its admin API, covering request rates, latency, error counts, and per-user/bucket statistics.

## Enabling Prometheus Metrics

Ceph RGW exposes metrics on a dedicated port. Enable the Prometheus exporter:

```bash
ceph config set client.rgw rgw_enable_ops_log false
ceph config set mgr mgr/prometheus/server_addr 0.0.0.0
ceph config set mgr mgr/prometheus/server_port 9283

# Enable the Prometheus module
ceph mgr module enable prometheus
```

The Prometheus endpoint is available at `http://your-mgr-host:9283/metrics`.

## Key RGW Metrics

| Metric | Description |
|--------|-------------|
| `ceph_rgw_req` | Total requests per RGW |
| `ceph_rgw_failed_req` | Failed requests |
| `ceph_rgw_get` | GET requests |
| `ceph_rgw_put` | PUT requests |
| `ceph_rgw_get_b` | Bytes downloaded |
| `ceph_rgw_put_b` | Bytes uploaded |
| `ceph_rgw_qlen` | Request queue length |
| `ceph_rgw_qactive` | Active requests |

## Prometheus Scrape Configuration

```yaml
scrape_configs:
  - job_name: 'ceph'
    static_configs:
      - targets: ['ceph-mgr.example.com:9283']
    scrape_interval: 30s
```

## Grafana Dashboard

Import the official Ceph RGW dashboard (ID: 5336) from Grafana.com or use the Rook-provided dashboards:

```bash
# In Rook with the monitoring stack enabled
kubectl get configmap -n rook-ceph | grep dashboard
```

Key panels to monitor:
- RGW request rate (GET/PUT/DELETE over time)
- Average request latency (p50, p95, p99)
- Error rate (5xx, 4xx responses)
- Active connections
- Bytes in/out per second

## Using the RGW Admin Stats API

Get overall RGW usage:

```bash
radosgw-admin usage show --show-log-entries=false
```

Get per-user stats:

```bash
radosgw-admin user stats --uid myuser --sync-stats
```

Get per-bucket stats:

```bash
radosgw-admin bucket stats --bucket mybucket
```

## Alerting Rules

Add Prometheus alert rules for RGW health:

```yaml
groups:
  - name: ceph-rgw
    rules:
      - alert: RGWHighErrorRate
        expr: rate(ceph_rgw_failed_req[5m]) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High RGW error rate detected"

      - alert: RGWQueueFull
        expr: ceph_rgw_qlen > 100
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "RGW request queue is backing up"
```

## Summary

Ceph RGW metrics are available via Prometheus through the Ceph MGR module, covering request rates, latency, and error counts. Use Grafana dashboards for visualization and Prometheus alert rules for proactive notification. Supplement with `radosgw-admin` commands for per-user and per-bucket usage reporting.
