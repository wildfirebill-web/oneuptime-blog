# How to Configure Rate Limits for RGW Buckets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Rate Limit, Bucket, Object Storage, Throttling, QoS

Description: Configure per-bucket rate limits in Ceph RGW to control request throughput on shared or high-traffic buckets regardless of which user is accessing them.

---

In addition to per-user rate limits, Ceph RGW supports per-bucket rate limits that throttle all access to a specific bucket, regardless of which user is making the requests. This is useful for shared buckets or buckets that need strict throughput controls.

## Why Bucket-Level Rate Limits

Bucket-level rate limits are useful when:
- A shared bucket is accessed by many users and you want to cap total throughput
- A bucket hosts content for an external system with known capacity limits
- You want to prevent a specific bucket from overloading storage backends during peak load

## Setting Bucket Rate Limits

```bash
# Set read and write limits on a specific bucket
radosgw-admin ratelimit set \
  --ratelimit-scope bucket \
  --bucket shared-uploads \
  --max-read-ops 2000 \
  --max-read-bytes 209715200 \
  --max-write-ops 500 \
  --max-write-bytes 52428800
```

Enable the bucket rate limit:

```bash
radosgw-admin ratelimit enable \
  --ratelimit-scope bucket \
  --bucket shared-uploads
```

## Getting Bucket Rate Limit Settings

```bash
radosgw-admin ratelimit get \
  --ratelimit-scope bucket \
  --bucket shared-uploads
```

Output:

```json
{
  "enabled": true,
  "max_read_ops": 2000,
  "max_read_bytes": 209715200,
  "max_write_ops": 500,
  "max_write_bytes": 52428800
}
```

## Combining User and Bucket Rate Limits

When both user and bucket rate limits are active, the more restrictive limit applies. For example:

- User alice has `max_read_ops=5000`
- Bucket `shared-uploads` has `max_read_ops=2000`
- Alice's effective read limit on `shared-uploads` is `2000`

This allows you to create broad user-level tiers and then further restrict specific high-risk buckets.

## Rate Limiting a Public or CDN-Origin Bucket

For a bucket used as a CDN origin:

```bash
# Allow high read throughput but very limited writes
radosgw-admin ratelimit set \
  --ratelimit-scope bucket \
  --bucket cdn-origin \
  --max-read-ops 50000 \
  --max-read-bytes 5368709120 \
  --max-write-ops 100 \
  --max-write-bytes 10485760

radosgw-admin ratelimit enable \
  --ratelimit-scope bucket \
  --bucket cdn-origin
```

## Auditing All Bucket Rate Limits

List all buckets and check which have rate limits:

```bash
#!/bin/bash
radosgw-admin metadata list bucket | jq -r '.[]' | while read BUCKET; do
  STATUS=$(radosgw-admin ratelimit get \
    --ratelimit-scope bucket --bucket "$BUCKET" 2>/dev/null | jq -r '.enabled')
  if [ "$STATUS" = "true" ]; then
    echo "Rate limited: $BUCKET"
  fi
done
```

## Disabling a Bucket Rate Limit

```bash
radosgw-admin ratelimit disable \
  --ratelimit-scope bucket \
  --bucket shared-uploads
```

## Summary

Bucket-level rate limits in Ceph RGW provide aggregate throttling for all access to a specific bucket, regardless of user identity. Use them for shared buckets, CDN origins, or any bucket that needs guaranteed throughput caps. When combined with user limits, the most restrictive of the two applies, allowing layered QoS policies.
