# How to Handle Disk Full Errors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Disk, Error Handling, Operations, Troubleshooting

Description: Learn how to diagnose, recover from, and prevent disk full errors in ClickHouse, including emergency space recovery and long-term capacity planning.

---

ClickHouse disk full errors can cause inserts to fail, merges to stall, and in severe cases, the server to crash. This guide covers diagnosing the problem, emergency recovery, and preventing recurrence.

## Identifying the Problem

When disk space is exhausted, ClickHouse logs errors like:

```text
DB::Exception: Not enough free disk space to write 4.00 GiB
DB::Exception: Too many parts (300). Merges are processing slower than inserts.
```

Check current disk usage immediately:

```sql
SELECT name, path,
    formatReadableSize(free_space) AS free_space,
    formatReadableSize(total_space) AS total_space,
    round((1 - free_space / total_space) * 100, 1) AS used_pct
FROM system.disks
ORDER BY free_space;
```

## Finding What Is Using Space

```sql
-- Find largest tables
SELECT
    database,
    table,
    formatReadableSize(sum(bytes_on_disk)) AS size,
    sum(rows) AS rows,
    count() AS parts
FROM system.parts
WHERE active = 1
GROUP BY database, table
ORDER BY sum(bytes_on_disk) DESC
LIMIT 20;

-- Check detached parts (orphaned disk usage)
SELECT
    database,
    table,
    formatReadableSize(sum(bytes_on_disk)) AS detached_size
FROM system.detached_parts
GROUP BY database, table
ORDER BY sum(bytes_on_disk) DESC;

-- Check shadow/freeze leftovers
SELECT formatReadableSize(total_size) AS shadow_size
FROM system.disks
WHERE name = 'default';
```

Check filesystem directly:

```bash
du -sh /var/lib/clickhouse/*/
du -sh /var/lib/clickhouse/shadow/
du -sh /var/lib/clickhouse/store/*/detached/
```

## Emergency Space Recovery

**Step 1: Remove old frozen backups (shadows)**

```sql
-- Unfreeze named backups
SYSTEM UNFREEZE WITH NAME 'old-backup-name';
```

```bash
# Or manually remove old shadow directories
ls -la /var/lib/clickhouse/shadow/
rm -rf /var/lib/clickhouse/shadow/1/  # Only if no restore is in progress
```

**Step 2: Drop detached parts**

```sql
-- Drop all detached parts from a table
ALTER TABLE analytics.events DROP DETACHED PART ALL;
```

**Step 3: Drop old partitions**

If you have TTL-expired data that hasn't been cleaned up:

```sql
-- Force TTL enforcement
ALTER TABLE analytics.events
MODIFY TTL event_time + INTERVAL 90 DAY DELETE;

ALTER TABLE analytics.events MATERIALIZE TTL;
```

**Step 4: Drop unused tables or databases**

```sql
DROP TABLE analytics.old_staging_table;
```

## Preventing Disk Full Errors

Set alerts before disk fills up using ClickHouse system metrics:

```sql
-- Create a view to monitor disk usage
CREATE VIEW system.disk_usage_alert AS
SELECT name, free_space, total_space,
    round((1 - free_space / total_space) * 100, 1) AS used_pct
FROM system.disks
WHERE used_pct > 80;
```

Integrate with your monitoring system (Prometheus, Grafana, OneUptime) to alert at 80% and 90% usage.

## Configuring Disk Reservation

Tell ClickHouse to reserve space and stop accepting writes before the disk fills completely:

```xml
<clickhouse>
  <storage_configuration>
    <disks>
      <default>
        <keep_free_space_bytes>10737418240</keep_free_space_bytes>
      </default>
    </disks>
  </storage_configuration>
</clickhouse>
```

This keeps 10GB reserved, preventing ClickHouse from filling the disk completely.

## Summary

ClickHouse disk full errors require immediate triage: check which tables consume the most space, remove orphaned shadow and detached parts, and enforce TTL deletion. Prevent recurrence by setting disk reservation via `keep_free_space_bytes`, enabling monitoring alerts at 80% usage, and implementing TTL policies on large tables. Long-term solutions include adding more storage, tiering to object storage, or increasing data compression.
