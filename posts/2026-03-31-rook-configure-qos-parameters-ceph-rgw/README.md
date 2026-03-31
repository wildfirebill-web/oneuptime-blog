# How to Configure QoS Parameters for Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, QoS, Object Storage

Description: Learn how to configure quality-of-service parameters for Ceph RGW including request rate limits, per-user throttling, and bucket-level QoS controls.

---

## RGW QoS Overview

Ceph RADOS Gateway (RGW) supports multiple layers of QoS to control object storage workloads:

- **Per-user rate limits** - Cap S3/Swift API requests per second per user
- **Per-bucket rate limits** - Limit throughput on specific buckets
- **Ratelimit zone** - Apply rate limits across all users in a zone
- **mClock backend limits** - Control OSD-level I/O from RGW operations

## Enabling Per-User Rate Limits

Set request rate limits on an individual RGW user:

```bash
# Limit to 1000 ops/minute and 1GB/hour per user
radosgw-admin user modify --uid=alice \
  --max-buckets=100 \
  --rate-limit-enabled=true \
  --max-read-ops=500 \
  --max-write-ops=200 \
  --max-read-bytes=536870912 \    # 512MB/s
  --max-write-bytes=268435456     # 256MB/s
```

Check the current limits:

```bash
radosgw-admin user info --uid=alice | python3 -m json.tool | grep -A10 rate
```

## Enabling Per-Bucket Rate Limits

Apply rate limits to a specific bucket:

```bash
radosgw-admin bucket limit set \
  --bucket=production-assets \
  --uid=alice \
  --max-read-ops=200 \
  --max-write-ops=50 \
  --max-read-bytes=104857600 \   # 100MB/s
  --max-write-bytes=52428800     # 50MB/s
```

View bucket limits:

```bash
radosgw-admin bucket limit get --bucket=production-assets
```

## Global Rate Limiting with ratelimit Zone

Apply global limits in the zone configuration:

```bash
radosgw-admin zone modify \
  --rgw-zone=default \
  --max-read-ops=10000 \
  --max-write-ops=5000
```

These limits apply to all users in the zone unless overridden at the user level.

## Configuring RGW Thread Pool Limits

Limit the number of concurrent RGW operations by controlling thread pools:

```bash
cat >> /etc/ceph/ceph.conf << 'EOF'
[client.rgw.gateway1]
rgw_thread_pool_size = 256
rgw_num_rados_handles = 8
rgw_max_chunk_size = 4194304
EOF
```

Apply configuration:

```bash
systemctl restart ceph-radosgw@rgw.gateway1
```

## Setting S3 API-Level Quotas

Configure storage quotas to indirectly enforce QoS:

```bash
# Per-user storage quota
radosgw-admin quota set --quota-scope=user \
  --uid=alice \
  --max-objects=100000 \
  --max-size=107374182400   # 100GB

# Per-bucket storage quota
radosgw-admin quota set --quota-scope=bucket \
  --uid=alice \
  --bucket=uploads \
  --max-objects=10000 \
  --max-size=10737418240   # 10GB

# Enable quotas
radosgw-admin quota enable --quota-scope=user --uid=alice
radosgw-admin quota enable --quota-scope=bucket --uid=alice --bucket=uploads
```

## Monitoring RGW QoS Metrics

Check user and bucket statistics:

```bash
radosgw-admin user stats --uid=alice
radosgw-admin bucket stats --bucket=production-assets
```

View throttle counters from the Ceph admin socket:

```bash
ceph daemon client.rgw.gateway1 perf dump | python3 -m json.tool | grep -i throttle
```

## Summary

Configuring QoS for Ceph RGW provides multi-layered protection against individual users or buckets overwhelming the object storage backend. Per-user and per-bucket rate limits control API request rates and bandwidth, while storage quotas prevent unlimited growth. These controls are essential for multi-tenant RGW deployments where fair resource sharing is required.
