# How to Track and Trim Usage Data in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Usage, Billing, Object Storage, Administration, Maintenance

Description: Track per-user and per-bucket usage data in Ceph RGW for billing and reporting, and trim old usage log entries to prevent unbounded storage growth.

---

Ceph RGW records detailed usage data for every S3 and Swift operation, including bytes sent and received per user and per bucket. This data is valuable for billing, chargeback reporting, and capacity planning. However, usage logs grow indefinitely unless trimmed periodically.

## Enabling Usage Logging

Usage logging must be explicitly enabled in RGW:

```bash
ceph config set client.rgw rgw_enable_usage_log true
```

Restart RGW for the change to take effect:

```bash
# Baremetal
sudo systemctl restart ceph-radosgw@rgw.$(hostname -s)

# Rook
kubectl rollout restart deployment -n rook-ceph -l app=rook-ceph-rgw
```

## Viewing Usage Data

Show all usage for all users across all time:

```bash
radosgw-admin usage show
```

Show usage for a specific user:

```bash
radosgw-admin usage show --uid alice
```

Show usage for a specific time range:

```bash
radosgw-admin usage show \
  --uid alice \
  --start-date 2026-03-01 \
  --end-date 2026-03-31
```

## Understanding Usage Output

```json
{
  "entries": [
    {
      "user": "alice",
      "buckets": [
        {
          "bucket": "mybucket",
          "time": "2026-03-31T10:00:00.000Z",
          "categories": [
            {
              "category": "get_obj",
              "bytes_sent": 10485760,
              "bytes_received": 0,
              "ops": 500,
              "successful_ops": 498
            },
            {
              "category": "put_obj",
              "bytes_sent": 0,
              "bytes_received": 52428800,
              "ops": 100,
              "successful_ops": 100
            }
          ]
        }
      ]
    }
  ]
}
```

Key categories: `get_obj`, `put_obj`, `delete_obj`, `list_bucket`, `create_bucket`, `delete_bucket`

## Showing Usage Without Log Entries

For a summary without the full log detail:

```bash
radosgw-admin usage show --show-log-entries=false
```

## Trimming Old Usage Logs

Usage logs accumulate over time. Trim them to prevent unbounded storage growth:

```bash
# Trim all usage data older than a specific date
radosgw-admin usage trim --end-date 2026-01-01

# Trim all usage data for a specific user
radosgw-admin usage trim --uid alice

# Trim for a user up to a specific date
radosgw-admin usage trim \
  --uid alice \
  --end-date 2026-02-28
```

## Automating Monthly Usage Reports and Trim

```bash
#!/bin/bash
# Run at end of month to export and trim usage

MONTH=$(date -d "last month" '+%Y-%m')
START="${MONTH}-01"
END=$(date -d "${MONTH}-01 +1 month" '+%Y-%m-%d')

# Export usage report to JSON
radosgw-admin usage show \
  --start-date "$START" \
  --end-date "$END" \
  > "/var/reports/rgw-usage-${MONTH}.json"

echo "Usage report saved for $MONTH"

# Trim logs older than the exported period
radosgw-admin usage trim --end-date "$START"
echo "Trimmed usage logs before $START"
```

## Summary

Ceph RGW usage tracking provides per-user, per-bucket, per-operation metrics for billing and capacity reporting. Enable it with `rgw_enable_usage_log`, query with `usage show`, and trim periodically with `usage trim` to control log storage growth. Combine monthly exports with automated trim jobs to maintain a complete billing history without unbounded pool growth.
