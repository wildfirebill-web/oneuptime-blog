# How to Monitor Bucket Size and Object Count in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, RGW, Monitoring, Object Storage, Quota

Description: Use radosgw-admin and Ceph CLI tools to monitor bucket size, object counts, and per-user storage consumption in Ceph RGW.

---

## Why Monitor Bucket Usage?

Tracking bucket size and object counts helps you enforce quotas, detect runaway data growth, plan capacity, and identify individual tenants consuming excessive storage.

## Checking a Single Bucket's Stats

Use `radosgw-admin` to get detailed statistics for a specific bucket:

```bash
radosgw-admin bucket stats --bucket my-bucket
```

Sample output:

```json
{
  "bucket": "my-bucket",
  "num_objects": 1523,
  "size": 4831838208,
  "size_actual": 4831838208,
  "size_kb": 4718592,
  "size_kb_actual": 4718600
}
```

The `size` field is in bytes. Divide by 1073741824 for GB.

## Listing All Buckets with Usage

Get a summary of all buckets:

```bash
radosgw-admin bucket list
```

For stats on all buckets at once:

```bash
radosgw-admin bucket stats
```

This outputs a JSON array with stats for every bucket in the cluster.

## Per-User Storage Usage

Check how much storage a specific user consumes:

```bash
radosgw-admin user stats --uid=alice
```

Enable usage tracking if not already on (required for per-user stats):

```bash
ceph config set client.rgw rgw_enable_usage_log true
systemctl restart ceph-radosgw@rgw.myzone
```

Then retrieve usage logs:

```bash
radosgw-admin usage show --uid=alice --start-date=2026-03-01 --end-date=2026-03-31
```

## Syncing Bucket Stats

RGW bucket stats can drift over time. Force a resync:

```bash
radosgw-admin bucket check --bucket my-bucket --fix
```

## Querying via Prometheus

If you have the Ceph MGR Prometheus module enabled, bucket metrics are exposed:

```bash
curl http://mgr-host:9283/metrics | grep rgw_bucket
```

Relevant metrics include:
- `ceph_rgw_req` - total RGW requests
- `ceph_rgw_get_b` - bytes served via GET

For per-bucket dashboards, use the Ceph RGW Grafana dashboard from the community repository.

## Scripting a Usage Report

Generate a simple CSV report of all bucket sizes:

```bash
radosgw-admin bucket stats | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
print('bucket,objects,size_gb')
for b in data:
    gb = b.get('usage', {}).get('rgw.main', {}).get('size', 0) / 1073741824
    objs = b.get('usage', {}).get('rgw.main', {}).get('num_objects', 0)
    print(f\"{b['bucket']},{objs},{gb:.2f}\")
"
```

## Summary

Use `radosgw-admin bucket stats` to inspect per-bucket object counts and sizes, and `radosgw-admin user stats` to track per-user consumption. Enable usage logging for detailed reporting, and use the Prometheus MGR module to integrate bucket metrics into Grafana dashboards. Periodically run `bucket check --fix` to resync stats after inconsistencies.
