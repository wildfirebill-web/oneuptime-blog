# How to Set Bucket Quotas for Individual Buckets in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, RGW, Quota, Object Storage, Capacity

Description: Configure per-bucket storage quotas in Ceph RGW to limit maximum object count and total byte size for individual buckets.

---

## Why Use Per-Bucket Quotas?

Per-bucket quotas allow you to cap the amount of data or the number of objects stored in a specific bucket. This is useful for multi-tenant environments, cost control, and preventing runaway storage growth from a single application.

## Setting a Quota on a Bucket

Use `radosgw-admin` to set quotas directly on a bucket. You can limit by size, object count, or both:

```bash
# Limit bucket to 10 GB and 1 million objects
radosgw-admin quota set \
  --bucket my-bucket \
  --quota-scope=bucket \
  --max-size=10737418240 \
  --max-objects=1000000
```

The `--max-size` value is in bytes. Use `-1` to disable a particular limit while keeping the other active.

## Enabling the Quota

Setting a quota does not automatically enable it. You must explicitly enable it:

```bash
radosgw-admin quota enable \
  --bucket my-bucket \
  --quota-scope=bucket
```

## Viewing Current Quota Settings

```bash
radosgw-admin bucket stats --bucket my-bucket
```

Look for the `bucket_quota` field in the output:

```json
"bucket_quota": {
  "enabled": true,
  "check_on_raw": false,
  "max_size": 10737418240,
  "max_size_kb": 10485760,
  "max_objects": 1000000
}
```

## Disabling a Quota

```bash
radosgw-admin quota disable \
  --bucket my-bucket \
  --quota-scope=bucket
```

## Setting User-Level Quotas

In addition to per-bucket quotas, you can set aggregate quotas per user:

```bash
radosgw-admin quota set \
  --uid=alice \
  --quota-scope=user \
  --max-size=107374182400 \
  --max-objects=10000000

radosgw-admin quota enable --uid=alice --quota-scope=user
```

## Testing Quota Enforcement

Upload objects until the limit is hit. Once exceeded, RGW returns:

```text
An error occurred (QuotaExceeded) when calling the PutObject operation:
Quota Exceeded
```

You can verify the current usage against the quota:

```bash
radosgw-admin bucket stats --bucket my-bucket | python3 -c "
import json, sys
d = json.load(sys.stdin)
usage = d.get('usage', {}).get('rgw.main', {})
quota = d.get('bucket_quota', {})
print(f\"Used: {usage.get('size',0)} bytes / Max: {quota.get('max_size','unlimited')}\")
print(f\"Objects: {usage.get('num_objects',0)} / Max: {quota.get('max_objects','unlimited')}\")
"
```

## Summary

Ceph RGW per-bucket quotas are set with `radosgw-admin quota set --quota-scope=bucket` and must be explicitly enabled with `quota enable`. Use `bucket stats` to view quota configuration and current usage. Combine with user-level quotas for complete multi-tenant storage governance.
