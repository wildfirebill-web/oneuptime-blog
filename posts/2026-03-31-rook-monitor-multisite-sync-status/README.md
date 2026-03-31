# How to Monitor Multisite Sync Status in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Multisite, Monitoring, Sync

Description: Learn how to monitor Ceph RGW multisite sync status using radosgw-admin commands, Prometheus metrics, and automated lag alerting for production object storage.

---

## Key Sync Status Metrics

For RGW multisite, you need to track:
- **Sync lag**: how far behind the secondary zone is from the primary
- **Error rate**: failed sync operations
- **Throughput**: objects and bytes replicated per second
- **Pending objects**: queue depth for replication

## Step 1: Check Overall Sync Status

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin sync status
```

Sample output to interpret:
```text
realm mycompany (realm-id)
    zonegroup us (zg-id)
        zone us-west (zone-id)
            data sync source: us-east (zone-id)
                              behind shards: 0
                              newest full sync: 2026-03-31 12:00:00
                              oldest incremental sync: 2026-03-31 12:05:00
```

`behind shards: 0` means fully synced. Non-zero means lag exists.

## Step 2: Check Per-Bucket Sync Status

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin bucket sync status \
  --bucket=my-critical-bucket \
  --source-zone=us-east

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin bucket sync markers \
  --bucket=my-critical-bucket
```

## Step 3: Check Sync Error Log

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin sync error list \
  --start-time=2026-03-30T00:00:00 \
  --end-time=2026-03-31T23:59:59 | python3 -m json.tool
```

For retry of failed syncs:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin sync error trim \
  --start-time=2026-03-30T00:00:00
```

## Step 4: Prometheus Metrics for RGW Sync

Enable RGW metrics:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_enable_ops_log true
```

Key Prometheus metrics from the RGW:

```text
# Scrape from http://rgw-service:7480/metrics
rgw_sync_seconds_behind - lag in seconds
rgw_data_sync_status - number of shards behind
rgw_metadata_sync_full_total - total full sync operations
```

## Step 5: Automated Lag Monitoring Script

```bash
#!/bin/bash
MAX_LAG_SHARDS=10
BEHIND=$(kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin sync status 2>&1 | \
  grep "behind shards" | awk '{print $NF}')

if [ "$BEHIND" -gt "$MAX_LAG_SHARDS" ]; then
  echo "ALERT: RGW sync lag - $BEHIND shards behind"
  # Send to PagerDuty/Slack here
  exit 1
fi
echo "OK: RGW sync lag = $BEHIND shards"
```

## Step 6: Prometheus Alert Rule

```yaml
groups:
- name: rgw-multisite
  rules:
  - alert: RGWSyncLagHigh
    expr: rgw_sync_seconds_behind > 300
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "RGW multisite sync lag exceeds 5 minutes"
      description: "Zone {{ $labels.zone }} is {{ $value }}s behind"

  - alert: RGWSyncStopped
    expr: increase(rgw_data_sync_status[10m]) == 0 and rgw_sync_seconds_behind > 0
    for: 15m
    labels:
      severity: critical
    annotations:
      summary: "RGW multisite sync appears stopped"
```

## Summary

Monitoring RGW multisite sync requires checking shard lag via `radosgw-admin sync status`, inspecting per-bucket sync markers, reviewing the error log for failed operations, and alerting on high lag or stopped sync via Prometheus. Regular monitoring ensures replication stays within acceptable recovery point objectives.
