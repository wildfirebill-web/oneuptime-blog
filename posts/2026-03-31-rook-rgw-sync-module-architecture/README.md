# How to Understand Ceph RGW Sync Module Architecture

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Sync, Architecture

Description: Learn how Ceph RGW sync modules work, including the data flow from bucket notifications to sync module processing for multi-zone and external system integration.

---

## Overview

Ceph RGW sync modules extend the multisite replication framework to forward object operations to external systems. Rather than simply replicating to another Ceph zone, a sync module can forward object metadata and data to Elasticsearch, a cloud provider, a message queue, or a custom endpoint. Understanding the architecture is essential for both using existing sync modules and writing new ones.

## Core Architecture Concepts

A Ceph RGW multisite deployment consists of realm, zonegroup, and zone objects. Sync modules operate at the zone level and intercept data sync operations before they are written to RADOS.

```bash
# View the multisite hierarchy
radosgw-admin realm list
radosgw-admin zonegroup list
radosgw-admin zone list

# Check sync status for a zone
radosgw-admin sync status

# View zone configuration including sync module type
radosgw-admin zone get --rgw-zone=secondary-zone | jq '.tier_config'
```

## Sync Module Components

```
Object Write (Primary Zone)
       |
       v
Bucket Index Log Entry
       |
       v
Data Sync Thread (per-shard)
       |
       v
RGWDataSyncCR (coroutine)
       |
       v
+------------------+
| Sync Module Hook |  <-- This is where custom logic runs
+------------------+
       |
       v
External System (ES, S3, custom)
```

The sync thread reads from the bucket index shard log, creates per-object sync entries, and calls the sync module's data handler for each operation.

## Step 1 - View Available Sync Module Types

```bash
# Current built-in sync modules
# - cloud   : sync to AWS S3, GCS, Azure
# - elasticsearch : index objects in Elasticsearch
# - pubsub  : send bucket notifications to topics
# - log     : write sync operations to a log

# Check zone tier type
radosgw-admin zone get --rgw-zone=my-zone | jq '.tier_type'
```

## Step 2 - Understand the Data Flow in Detail

```bash
# Monitor sync datalog in real time
radosgw-admin datalog list --max-entries=10

# Check per-shard sync position
radosgw-admin sync status --rgw-zone=secondary-zone

# The sync process works as follows:
# 1. Primary zone writes object -> updates bucket index log
# 2. Secondary zone's data sync thread polls the datalog
# 3. For each modified object, the sync module handler is called
# 4. The handler can: copy the object, call an API, write metadata
```

## Step 3 - Configure Sync Threads and Performance

```bash
# Adjust sync thread count for higher throughput
ceph config set client.rgw.my-store rgw_data_sync_concurrency 32
ceph config set client.rgw.my-store rgw_bucket_sync_spawn_window 20

# Adjust polling interval
ceph config set client.rgw.my-store rgw_data_sync_poll_interval 5

# Check current sync lag
radosgw-admin sync status 2>&1 | grep -A5 "data sync"
```

## Step 4 - Inspect Sync Module Configuration

```bash
# View the full zone configuration for a sync module
radosgw-admin zone get --rgw-zone=elasticsearch-zone | jq '{
  name: .name,
  tier_type: .tier_type,
  tier_config: .tier_config
}'

# Example tier_config for Elasticsearch:
# {
#   "tier_type": "elasticsearch",
#   "tier_config": {
#     "endpoint": "http://elasticsearch:9200",
#     "num_shards": 16,
#     "num_replicas": 1,
#     "index_buckets_list": [],
#     "explicit_custom_meta": false
#   }
# }
```

## Step 5 - Monitor Sync Module Health

```bash
# Check for sync errors
radosgw-admin sync error list --max-entries=20

# Get the sync status for all shards
radosgw-admin sync status | grep -E "(behind|error|shard)"

# Check Prometheus metrics for sync lag
curl -s http://rgw.example.com:9283/metrics | grep rgw_sync
```

## Step 6 - Understand Bucket Notifications vs Sync Modules

```bash
# Bucket notifications are a separate mechanism
# - Triggered per-operation (PUT, DELETE, etc.)
# - Push-based (HTTP, Kafka, AMQP)
# Sync modules operate on the datalog pull model

# List active bucket notifications
radosgw-admin pubsub topics list

# Create a bucket notification (different from sync module)
aws --endpoint-url http://rgw.example.com:7480 \
  s3api put-bucket-notification-configuration \
  --bucket my-bucket \
  --notification-configuration file://notification.json
```

## Summary

Ceph RGW sync modules operate within the multisite replication framework, intercepting data sync operations at the zone level to forward object data and metadata to external systems. The pull-based datalog model ensures reliable delivery even when the external system is temporarily unavailable. Understanding the hierarchy of realm, zonegroup, and zone objects is foundational before implementing any custom or built-in sync module.
