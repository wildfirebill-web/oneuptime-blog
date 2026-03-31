# How to Configure Read/Write Rate Limits for RGW Users

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Rate Limit, User Management, Object Storage, Throttling, QoS

Description: Configure separate read and write rate limits for individual Ceph RGW users to provide differentiated quality of service and prevent resource monopolization.

---

Ceph RGW allows setting separate rate limits for read and write operations on a per-user basis. This enables differentiated Quality of Service (QoS), where some users get higher throughput allowances than others based on their tier or use case.

## Read vs Write Rate Limit Parameters

For user-level rate limits, you can configure:

| Parameter | Description |
|-----------|-------------|
| `--max-read-ops` | Max GET/HEAD/LIST ops per minute |
| `--max-write-ops` | Max PUT/DELETE/COPY ops per minute |
| `--max-read-bytes` | Max bytes downloaded per minute |
| `--max-write-bytes` | Max bytes uploaded per minute |

## Configuring a Read-Heavy User (Analytics)

An analytics user that reads frequently but rarely writes:

```bash
radosgw-admin ratelimit set \
  --ratelimit-scope user \
  --uid analytics-user \
  --max-read-ops 10000 \
  --max-read-bytes 1073741824 \
  --max-write-ops 100 \
  --max-write-bytes 10485760

radosgw-admin ratelimit enable \
  --ratelimit-scope user \
  --uid analytics-user
```

## Configuring a Write-Heavy User (Ingestion)

A data ingestion user that writes frequently but rarely reads:

```bash
radosgw-admin ratelimit set \
  --ratelimit-scope user \
  --uid ingestion-user \
  --max-read-ops 500 \
  --max-read-bytes 52428800 \
  --max-write-ops 5000 \
  --max-write-bytes 524288000

radosgw-admin ratelimit enable \
  --ratelimit-scope user \
  --uid ingestion-user
```

## Setting Only Read Limits (No Write Restriction)

Set 0 to indicate no limit for a specific direction:

```bash
radosgw-admin ratelimit set \
  --ratelimit-scope user \
  --uid backup-user \
  --max-read-ops 0 \
  --max-read-bytes 0 \
  --max-write-ops 1000 \
  --max-write-bytes 104857600
```

A value of `0` means unlimited for that specific parameter.

## Bulk Applying Different Rate Limits by Group

Create a function to apply tier-based rate limits:

```bash
#!/bin/bash

apply_tier_limits() {
  local UID=$1
  local TIER=$2

  case $TIER in
    basic)
      radosgw-admin ratelimit set \
        --ratelimit-scope user --uid "$UID" \
        --max-read-ops 500 --max-write-ops 200 \
        --max-read-bytes 52428800 --max-write-bytes 20971520
      ;;
    standard)
      radosgw-admin ratelimit set \
        --ratelimit-scope user --uid "$UID" \
        --max-read-ops 2000 --max-write-ops 1000 \
        --max-read-bytes 209715200 --max-write-bytes 104857600
      ;;
    premium)
      radosgw-admin ratelimit set \
        --ratelimit-scope user --uid "$UID" \
        --max-read-ops 10000 --max-write-ops 5000 \
        --max-read-bytes 1073741824 --max-write-bytes 524288000
      ;;
  esac

  radosgw-admin ratelimit enable \
    --ratelimit-scope user --uid "$UID"
}

# Usage
apply_tier_limits alice premium
apply_tier_limits bob standard
apply_tier_limits charlie basic
```

## Verifying Rate Limits in Action

Upload a file and observe throttling in RGW logs:

```bash
# Enable a low write limit temporarily
radosgw-admin ratelimit set \
  --ratelimit-scope user --uid testuser \
  --max-write-ops 5

radosgw-admin ratelimit enable --ratelimit-scope user --uid testuser

# Try rapid uploads
for i in $(seq 1 20); do
  aws s3 cp /tmp/testfile.txt s3://mybucket/file$i.txt \
    --endpoint-url http://your-rgw-host:7480 2>&1 | grep -E "SlowDown|upload"
done
```

## Summary

Ceph RGW per-user read/write rate limits provide granular QoS control, allowing separate throttles for GET/list operations and PUT/delete operations. Use this to implement tiered service plans, protect against accidental data floods, and ensure fair resource sharing in multi-tenant environments. Values of 0 disable a specific limit while keeping others active.
