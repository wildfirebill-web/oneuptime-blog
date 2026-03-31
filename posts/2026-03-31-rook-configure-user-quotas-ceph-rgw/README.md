# How to Configure User Quotas in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Quota, User Management, Object Storage, Governance, Administration

Description: Configure user-level quotas in Ceph RGW to limit the total storage size and number of objects a user can consume across all their buckets.

---

Ceph RGW user quotas allow administrators to set limits on how much total storage capacity and how many objects each user can consume across all their buckets. When a user exceeds their quota, upload operations fail with a `QuotaExceeded` error.

## Understanding User Quota Scope

User quotas are the aggregate limit across all buckets owned by a user:
- `max_size`: Maximum total bytes across all the user's buckets
- `max_objects`: Maximum total number of objects across all buckets
- `-1` values mean unlimited

## Setting User Quotas

Set quotas on a new user at creation:

```bash
radosgw-admin user create \
  --uid alice \
  --display-name "Alice Smith" \
  --max-buckets 20
```

Set user quota on an existing user:

```bash
radosgw-admin quota set \
  --uid alice \
  --quota-scope user \
  --max-size 10737418240 \
  --max-objects 1000000
```

Here `--max-size` is in bytes. Equivalent values:
- 1 GB = 1073741824
- 10 GB = 10737418240
- 100 GB = 107374182400

## Using Size Suffixes

You can also specify size with suffixes when using the admin API directly, but with `radosgw-admin` use bytes:

```bash
# Set 5 GiB user quota
radosgw-admin quota set \
  --uid alice \
  --quota-scope user \
  --max-size $((5 * 1024 * 1024 * 1024)) \
  --max-objects -1
```

## Enabling Quotas

Setting quota values does not automatically enable them. Enable the user quota:

```bash
radosgw-admin quota enable \
  --uid alice \
  --quota-scope user
```

## Viewing User Quota Settings

```bash
radosgw-admin quota get \
  --uid alice \
  --quota-scope user
```

Output:

```json
{
  "enabled": true,
  "max_size": 10737418240,
  "max_objects": 1000000
}
```

## Checking Current Usage Against Quota

First sync stats to ensure they are up to date:

```bash
radosgw-admin user stats --uid alice --sync-stats
```

Then view usage:

```bash
radosgw-admin user info --uid alice | jq '{quota: .user_quota, stats: .stats}'
```

## What Happens When Quota Is Exceeded

When a user tries to upload an object that would exceed their quota, RGW returns:

```
HTTP/1.1 403 Forbidden
<Error>
  <Code>QuotaExceeded</Code>
  <Message>User quota exceeded</Message>
</Error>
```

## Summary

Ceph RGW user quotas enforce aggregate storage limits across all buckets owned by a user. Set quotas with `radosgw-admin quota set --quota-scope user`, then explicitly enable them. Always sync stats before checking usage to ensure accuracy. Quotas are enforced at upload time and return a 403 QuotaExceeded error when limits are reached.
