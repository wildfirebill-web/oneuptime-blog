# How to Set Sync Polling Intervals in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Multisite, Sync, Replication

Description: Configure sync polling intervals in Ceph RGW multisite to balance replication latency against resource consumption on source and destination zones.

---

In a Ceph RGW multisite setup, secondary zones poll the primary zone for data and metadata changes. The polling interval controls how frequently changes are replicated and the associated CPU/network overhead.

## Sync Polling Parameters

```bash
# Check current sync interval settings
ceph config get client.rgw rgw_sync_log_trim_interval
ceph config get client.rgw rgw_meta_sync_status_update_period
```

Key parameters:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `rgw_sync_lease_period` | 120 | Seconds to hold a sync shard lease |
| `rgw_sync_log_trim_interval` | 1200 | Seconds between sync log trimming |
| `rgw_meta_sync_status_update_period` | 30 | Metadata sync status update period |
| `rgw_data_sync_poll_interval` | 20 | Data log polling interval (seconds) |

## Configuring Data Sync Polling Interval

Reduce polling interval for lower replication latency:

```bash
# Poll every 5 seconds (more responsive, higher overhead)
ceph config set client.rgw rgw_data_sync_poll_interval 5

# Poll every 60 seconds (lower overhead, higher latency)
ceph config set client.rgw rgw_data_sync_poll_interval 60
```

## Configuring Metadata Sync Interval

```bash
# Update metadata sync status every 15 seconds
ceph config set client.rgw rgw_meta_sync_status_update_period 15

# Trim sync log every 10 minutes
ceph config set client.rgw rgw_sync_log_trim_interval 600
```

## Applying in Rook

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: |
    [client.rgw.my-store.a]
    rgw_data_sync_poll_interval = 20
    rgw_meta_sync_status_update_period = 30
    rgw_sync_log_trim_interval = 1200
```

Apply and restart RGW:

```bash
kubectl apply -f rook-config-override.yaml
kubectl -n rook-ceph rollout restart deployment rook-ceph-rgw-my-store-a
```

## Monitoring Sync Lag

After tuning, verify replication lag:

```bash
# Check overall sync status
radosgw-admin sync status

# Check data sync lag for a specific zone
radosgw-admin data sync status --source-zone=primary-zone

# Look for "behind" entries
radosgw-admin sync status | grep -i "behind\|error\|lag"
```

Sample output showing sync is current:

```bash
          data sync source: primary-zone/us-east-1
              source zone primary-zone
                full sync: 0/128 shards
                incremental sync: 128/128 shards
                data is caught up with source
```

## Trade-offs

| Setting | Pros | Cons |
|---------|------|------|
| Short interval (5s) | Low replication lag | Higher CPU/network on source |
| Long interval (60s) | Low overhead | Higher replication lag |

Choose based on your RPO (Recovery Point Objective) requirements.

## Summary

Sync polling intervals control the trade-off between replication latency and resource overhead in Ceph RGW multisite. Set `rgw_data_sync_poll_interval` to 5-20 seconds for near-real-time replication, or 30-60 seconds for a quieter background sync. Monitor lag with `radosgw-admin sync status` and adjust based on your RPO requirements.
