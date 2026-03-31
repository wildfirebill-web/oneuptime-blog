# How to Monitor Scrubbing Progress in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Scrubbing, Monitoring, Prometheus, Kubernetes, Data Integrity

Description: Learn how to monitor Ceph scrubbing progress, track which PGs have been scrubbed, identify overdue scrubs, and set up alerts for scrubbing health issues.

---

## Why Monitor Scrubbing Progress

Ceph scrubbing is a background process that periodically verifies data integrity. Without monitoring, you might not know that:
- Scrubs are not completing due to restricted time windows
- Specific PGs have not been scrubbed in weeks
- Scrub errors are accumulating silently
- Scrubbing is consuming excessive resources

## Checking Overall Scrub Status

Get a quick overview of scrubbing activity:

```bash
# Check current cluster status for scrubbing info
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph status | grep -E "scrub|pg"

# Check for health warnings related to scrubbing
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph health detail | grep -E "scrub|PG_NOT_SCRUBBED|PG_NOT_DEEP_SCRUBBED"
```

## Identifying Overdue PGs

Find PGs that haven't been scrubbed within the expected interval:

```bash
# List PGs with their last scrub timestamps
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg dump | awk '{print $1, $14, $15}' | head -30
# Columns: pgid, last_scrub, last_deep_scrub

# Find PGs not scrubbed in the last 7 days
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c '
CUTOFF=$(date -d "7 days ago" +%s 2>/dev/null || date -v-7d +%s)
ceph pg dump 2>/dev/null | awk -v cutoff=$CUTOFF "NR>1 && \$14 != \"\" {
  cmd = \"date -d \" \$14 \" +%s\"
  cmd | getline ts
  close(cmd)
  if (ts < cutoff) print \$1, \$14
}"'
```

## Watching Active Scrubs

Monitor which PGs are currently being scrubbed:

```bash
# Watch for active scrubs
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  watch -n 5 "ceph pg dump | grep -E 'scrubbing|deep'"

# Count currently scrubbing PGs
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg dump | grep -c "scrubbing"
```

## Prometheus Metrics for Scrubbing

Monitor scrubbing via Prometheus when Rook monitoring is enabled:

Key scrub-related Prometheus metrics:

| Metric | Description |
|---|---|
| `ceph_pg_scrubbing` | Number of PGs currently scrubbing |
| `ceph_pg_deep_scrubbing` | Number of PGs currently deep scrubbing |
| `ceph_pg_inconsistent` | Number of inconsistent PGs |
| `ceph_osd_scrub_error` | Total scrub errors per OSD |

Sample PromQL queries:

```promql
# Current scrubbing PGs
ceph_pg_scrubbing + ceph_pg_deep_scrubbing

# Inconsistent PGs alert threshold
ceph_pg_inconsistent > 0

# Scrub error rate per OSD
rate(ceph_osd_scrub_error[1h]) > 0
```

## Setting Up Scrub Health Alerts

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ceph-scrub-alerts
  namespace: rook-ceph
spec:
  groups:
  - name: ceph-scrub
    rules:
    - alert: CephPGInconsistent
      expr: ceph_pg_inconsistent > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "{{ $value }} Ceph PGs have scrub inconsistencies"

    - alert: CephScrubErrors
      expr: increase(ceph_osd_scrub_error[1h]) > 0
      labels:
        severity: warning
      annotations:
        summary: "OSD {{ $labels.ceph_daemon }} has scrub errors"
```

## Generating a Scrub Coverage Report

Track what percentage of PGs have been scrubbed recently:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c '
TOTAL=$(ceph pg stat | grep -oP "[0-9]+ pgs")
ACTIVE=$(ceph pg dump 2>/dev/null | awk "NR>1 {count++} END {print count}")
echo "Total PGs: $ACTIVE"
echo "PGs with overdue scrubs:"
ceph health detail | grep "not scrubbed\|not deep-scrubbed" | wc -l
'
```

## Summary

Monitoring Ceph scrubbing progress requires tracking three dimensions: active scrubs in progress, overdue PGs that need scrubbing, and scrub error/inconsistency counts. Use `ceph health detail` for immediate status and Prometheus metrics for long-term trending. Set up alerts for inconsistent PGs (critical) and scrub errors (warning), and periodically audit your PG scrub timestamps to ensure all PGs are being scrubbed within their configured maximum intervals.
