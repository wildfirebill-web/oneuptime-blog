# How to Configure Data and Metadata Logging in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Logging, Multisite, Object Storage

Description: Configure data and metadata change logging in Ceph RGW to enable multisite replication, auditing, and change tracking for objects and bucket metadata.

---

Ceph RGW maintains separate logs for data changes (object uploads/deletes) and metadata changes (bucket/user operations). These logs are the foundation of multisite replication and can also be used for auditing.

## Understanding Data and Metadata Logs

- **Data log** - Records per-shard changes to objects in buckets (used for data sync)
- **Metadata log** - Records changes to users, buckets, and zone configuration (used for metadata sync)

## Key Logging Parameters

```bash
# Check current settings
ceph config get client.rgw rgw_data_log_num_shards
ceph config get client.rgw rgw_md_log_max_shards
```

## Configuring Data Log Shards

Data logs are sharded for parallel processing:

```bash
# Number of data log shards (default: 128)
# Changing this requires resync of all zones
ceph config set client.rgw rgw_data_log_num_shards 128

# For very large deployments
ceph config set client.rgw rgw_data_log_num_shards 256
```

**Warning:** Changing `rgw_data_log_num_shards` after initial setup requires a full resync of all secondary zones.

## Configuring Metadata Log

```bash
# Maximum metadata log shards
ceph config set client.rgw rgw_md_log_max_shards 64
```

## Viewing Log Contents

```bash
# List data log entries
radosgw-admin datalog list

# List for specific shard
radosgw-admin datalog list --shard-id=0

# List metadata log
radosgw-admin mdlog list

# Trim processed log entries
radosgw-admin datalog trim --shard-id=0 --end-marker=<marker>
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
    rgw_data_log_num_shards = 128
    rgw_md_log_max_shards = 64
```

Apply and restart RGW:

```bash
kubectl apply -f rook-config-override.yaml
kubectl -n rook-ceph rollout restart deployment rook-ceph-rgw-my-store-a
```

## Checking Log Health in Multisite

```bash
# Check if logs are being consumed by secondary zones
radosgw-admin sync status

# Get data sync status for a specific source
radosgw-admin data sync status --source-zone=us-east-1

# Check for errors
radosgw-admin sync error list
```

## Log Trimming

Logs grow indefinitely if not trimmed. In a multisite setup, logs are auto-trimmed after zones acknowledge processing. In single-zone deployments:

```bash
# Manually trim data log
radosgw-admin datalog trim --start-date=2025-01-01 --end-date=2025-12-31
```

## Summary

RGW data and metadata logs are essential for multisite replication and can support auditing workflows. Configure `rgw_data_log_num_shards` before initial deployment and avoid changing it afterward. Monitor log health with `radosgw-admin sync status` and ensure logs are being trimmed to prevent unbounded growth.
