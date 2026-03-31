# How to Configure Bucket Quotas in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Quota, Bucket, Object Storage, Governance, Administration

Description: Configure bucket-level quotas in Ceph RGW to limit the maximum size and number of objects within individual buckets, independent of user quotas.

---

Bucket quotas in Ceph RGW set limits on individual buckets, independent of user-level quotas. A bucket quota restricts the maximum total size and object count within a single bucket, making it useful for multi-tenant environments where each bucket represents a project or customer.

## Bucket Quota vs User Quota

| Quota Type | Scope | Enforcement |
|-----------|-------|------------|
| User quota | All buckets owned by user | Per-user aggregate |
| Bucket quota | Single bucket | Per-bucket |

Both quota types can be active simultaneously. The most restrictive limit applies.

## Setting a Bucket Quota

Set quota on a specific bucket:

```bash
radosgw-admin quota set \
  --uid alice \
  --bucket mybucket \
  --quota-scope bucket \
  --max-size 5368709120 \
  --max-objects 500000
```

Note: `--uid` specifies the bucket owner.

## Enabling the Bucket Quota

```bash
radosgw-admin quota enable \
  --uid alice \
  --bucket mybucket \
  --quota-scope bucket
```

## Viewing Bucket Quota Settings

```bash
radosgw-admin quota get \
  --uid alice \
  --bucket mybucket \
  --quota-scope bucket
```

Output:

```json
{
  "enabled": true,
  "max_size": 5368709120,
  "max_size_kb": 5242880,
  "max_objects": 500000
}
```

## Checking Bucket Usage Against Quota

Sync and view bucket stats:

```bash
radosgw-admin bucket stats --bucket mybucket
```

Key fields in the output:

```json
{
  "bucket": "mybucket",
  "usage": {
    "rgw.main": {
      "size": 1073741824,
      "num_objects": 12500
    }
  },
  "bucket_quota": {
    "enabled": true,
    "max_size": 5368709120,
    "max_objects": 500000
  }
}
```

## Setting Quota Without Specifying Object Limit

Allow unlimited objects but cap storage size:

```bash
radosgw-admin quota set \
  --uid alice \
  --bucket mybucket \
  --quota-scope bucket \
  --max-size 10737418240 \
  --max-objects -1

radosgw-admin quota enable \
  --uid alice \
  --bucket mybucket \
  --quota-scope bucket
```

## Setting Quota at Bucket Creation

You can apply quota immediately after creating a bucket:

```bash
# Create the bucket
aws s3api create-bucket \
  --bucket project-data \
  --endpoint-url http://your-rgw-host:7480

# Set and enable quota
radosgw-admin quota set \
  --uid alice \
  --bucket project-data \
  --quota-scope bucket \
  --max-size 21474836480 \
  --max-objects -1

radosgw-admin quota enable \
  --uid alice \
  --bucket project-data \
  --quota-scope bucket
```

## Summary

Ceph RGW bucket quotas enforce per-bucket limits on total size and object count, independent of user quotas. Set the quota with `radosgw-admin quota set --quota-scope bucket`, enable it explicitly, and check current usage with `bucket stats`. Use bucket quotas to enforce per-project or per-tenant storage limits in shared environments.
