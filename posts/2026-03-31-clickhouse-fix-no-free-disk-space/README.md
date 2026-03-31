# How to Fix "No free disk space" Errors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Troubleshooting, Disk Space, Storage, Maintenance

Description: Resolve "No free disk space" errors in ClickHouse by identifying disk consumers, removing old data, configuring TTL, and expanding storage capacity.

---

When ClickHouse runs out of disk space, it stops accepting writes and may place tables in readonly mode. Quick action is needed to restore operations without data loss.

## Immediate Triage

Check disk usage:

```bash
df -h /var/lib/clickhouse
du -sh /var/lib/clickhouse/*
```

Check ClickHouse's view:

```sql
SELECT
    name,
    path,
    formatReadableSize(total_space) AS total,
    formatReadableSize(free_space) AS free,
    formatReadableSize(total_space - free_space) AS used
FROM system.disks;
```

## Finding the Biggest Tables

```sql
SELECT
    database,
    table,
    formatReadableSize(sum(bytes_on_disk)) AS size_on_disk,
    count() AS part_count
FROM system.parts
WHERE active = 1
GROUP BY database, table
ORDER BY sum(bytes_on_disk) DESC
LIMIT 20;
```

## Removing Inactive Data Parts

Inactive parts (from merges, mutations) consume space but are still retained for a period:

```sql
-- Check inactive parts
SELECT
    database,
    table,
    formatReadableSize(sum(bytes_on_disk)) AS inactive_size
FROM system.parts
WHERE active = 0
GROUP BY database, table
ORDER BY sum(bytes_on_disk) DESC;

-- Force remove inactive parts
ALTER TABLE mydb.my_table CLEANUP;
```

## Cleaning Up Old Logs

ClickHouse logs in `system.query_log`, `system.part_log`, etc. consume disk space:

```sql
-- Check log table sizes
SELECT
    name,
    formatReadableSize(total_bytes) AS size
FROM system.tables
WHERE database = 'system'
  AND engine LIKE '%MergeTree%'
ORDER BY total_bytes DESC;

-- Truncate log tables
TRUNCATE TABLE system.query_log;
TRUNCATE TABLE system.part_log;
TRUNCATE TABLE system.trace_log;
```

## Deleting Old Data

Delete old data partitions by date:

```sql
-- List partitions
SELECT partition, formatReadableSize(sum(bytes_on_disk)) AS size
FROM system.parts
WHERE active = 1 AND table = 'events'
GROUP BY partition
ORDER BY partition;

-- Drop old partitions
ALTER TABLE mydb.events DROP PARTITION '202301';
ALTER TABLE mydb.events DROP PARTITION '202302';
```

## Configuring TTL to Prevent Recurrence

Add a TTL to automatically remove old data:

```sql
ALTER TABLE mydb.events
MODIFY TTL event_time + INTERVAL 90 DAY;
```

For system log tables, configure retention in ClickHouse config:

```xml
<clickhouse>
  <query_log>
    <ttl>event_date + INTERVAL 30 DAY DELETE</ttl>
  </query_log>
  <part_log>
    <ttl>event_date + INTERVAL 7 DAY DELETE</ttl>
  </part_log>
</clickhouse>
```

## Expanding Storage

If the above steps are insufficient, expand storage:

```bash
# Resize EBS volume (AWS)
aws ec2 modify-volume --volume-id vol-xxx --size 2000

# Resize filesystem after volume expansion
sudo growpart /dev/xvdf 1
sudo xfs_growfs /var/lib/clickhouse
```

## Summary

"No free disk space" errors require immediate action: use `system.parts` to find large tables and inactive parts, drop old partitions, clean up system log tables, and add TTL policies to prevent recurrence. For long-term resolution, expand the storage volume or add cold storage tiers using ClickHouse's multi-disk storage policies.
