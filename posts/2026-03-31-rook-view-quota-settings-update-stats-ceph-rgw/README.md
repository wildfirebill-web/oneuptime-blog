# How to View Quota Settings and Update Stats in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Quota, Statistics, Object Storage, Administration, Monitoring

Description: View quota settings and synchronize usage statistics in Ceph RGW to ensure accurate quota enforcement and reporting across users and buckets.

---

Ceph RGW tracks usage statistics asynchronously, which means the stored stats may lag behind the actual current usage. To get accurate quota enforcement and reporting, you need to periodically synchronize (sync) stats. This guide covers how to view quota settings and update stats effectively.

## Why Stats Need to Be Synced

RGW maintains usage counters in a distributed cache and flushes them periodically. Commands like `user stats` or `bucket stats` may show stale numbers unless you explicitly trigger a sync. For quota enforcement, RGW uses the last-known stats, so stale stats can temporarily allow over-quota uploads.

## Viewing User Quota and Current Stats

Always sync before viewing to get accurate numbers:

```bash
# Sync user stats first
radosgw-admin user stats --uid alice --sync-stats

# View user info including quota and usage
radosgw-admin user info --uid alice
```

Look for the `stats` and `user_quota` sections in the output:

```json
{
  "user_id": "alice",
  "stats": {
    "size": 5368709120,
    "size_actual": 5368709120,
    "num_objects": 45000
  },
  "user_quota": {
    "enabled": true,
    "max_size": 10737418240,
    "max_objects": 1000000
  }
}
```

## Viewing Bucket Quota and Stats

```bash
# Get bucket stats with sync
radosgw-admin bucket stats --bucket mybucket
```

To force a stat update:

```bash
radosgw-admin bucket sync --bucket mybucket
radosgw-admin bucket stats --bucket mybucket
```

## Getting Quota Settings Only

```bash
# User quota
radosgw-admin quota get \
  --uid alice \
  --quota-scope user

# Bucket quota
radosgw-admin quota get \
  --uid alice \
  --bucket mybucket \
  --quota-scope bucket
```

## Syncing Stats for All Buckets of a User

```bash
# List all buckets owned by the user
BUCKETS=$(radosgw-admin bucket list --uid alice | jq -r '.[]')

for BUCKET in $BUCKETS; do
  echo "Syncing stats for $BUCKET"
  radosgw-admin bucket sync --bucket "$BUCKET"
done
```

## Global Stats Sync

To sync stats for all users (useful after a cluster restart or after fixing inconsistencies):

```bash
radosgw-admin user list | jq -r '.[]' | while read UID; do
  radosgw-admin user stats --uid "$UID" --sync-stats
done
```

## Monitoring Quota Usage Percentage

A script to check quota utilization:

```bash
#!/bin/bash
UID=$1
STATS=$(radosgw-admin user stats --uid "$UID" --sync-stats)
USED=$(echo "$STATS" | jq -r '.stats.size')
MAX=$(radosgw-admin quota get --uid "$UID" --quota-scope user | jq -r '.max_size')

if [ "$MAX" != "-1" ] && [ "$MAX" -gt 0 ]; then
  PERCENT=$(echo "scale=1; $USED * 100 / $MAX" | bc)
  echo "User $UID: ${PERCENT}% of quota used ($USED / $MAX bytes)"
else
  echo "User $UID: no quota limit set"
fi
```

## Summary

Accurate quota and usage reporting in Ceph RGW requires syncing stats with `--sync-stats` before reading them, since RGW flushes usage counters asynchronously. Use `quota get` for the configured limits and `user stats` or `bucket stats` for current consumption. Build periodic stat-sync jobs to keep usage reporting current and ensure quota enforcement is accurate.
