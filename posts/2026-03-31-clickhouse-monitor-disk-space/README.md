# How to Monitor Disk Space Usage in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Monitoring, Storage, Database, Administration

Description: Learn how to monitor disk space usage in ClickHouse using system tables, query disk and volume stats, and set up alerts for low free space conditions.

---

Running out of disk space causes ClickHouse to stop accepting inserts and can corrupt ongoing merges. Proactive disk monitoring is a critical part of operating a production ClickHouse cluster. ClickHouse exposes detailed disk and part-level statistics through its system tables, making it straightforward to build dashboards and alerting queries.

## Checking Overall Disk Usage

The `system.disks` table reports current free and total space for every configured disk:

```sql
SELECT
    name                                              AS disk,
    type,
    formatReadableSize(free_space)                    AS free,
    formatReadableSize(total_space)                   AS total,
    formatReadableSize(total_space - free_space)      AS used,
    round((1 - free_space / total_space) * 100, 2)   AS used_pct
FROM system.disks
ORDER BY used_pct DESC;
```

## Checking Space Per Table

```sql
SELECT
    database,
    table,
    formatReadableSize(sum(bytes_on_disk))          AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    count()                                          AS parts
FROM system.parts
WHERE active = 1
GROUP BY database, table
ORDER BY sum(bytes_on_disk) DESC
LIMIT 20;
```

## Checking Space Per Database

```sql
SELECT
    database,
    formatReadableSize(sum(bytes_on_disk))          AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    count()                                          AS parts
FROM system.parts
WHERE active = 1
GROUP BY database
ORDER BY sum(bytes_on_disk) DESC;
```

## Checking Space Per Disk Across Tables

See how data is distributed across physical disks:

```sql
SELECT
    disk_name,
    database,
    table,
    formatReadableSize(sum(bytes_on_disk)) AS size,
    count()                                AS parts
FROM system.parts
WHERE active = 1
GROUP BY disk_name, database, table
ORDER BY disk_name, sum(bytes_on_disk) DESC;
```

## Checking Space Per Partition

Identify which partitions are consuming the most space:

```sql
SELECT
    database,
    table,
    partition,
    disk_name,
    formatReadableSize(sum(bytes_on_disk))          AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed
FROM system.parts
WHERE active = 1
  AND database = currentDatabase()
GROUP BY database, table, partition, disk_name
ORDER BY sum(bytes_on_disk) DESC
LIMIT 30;
```

## Checking Inactive (Non-Merged) Parts

Inactive parts are parts that have been superseded by a merge but not yet deleted. They temporarily inflate disk usage:

```sql
SELECT
    database,
    table,
    active,
    count()                                AS parts,
    formatReadableSize(sum(bytes_on_disk)) AS size
FROM system.parts
GROUP BY database, table, active
ORDER BY database, table, active;
```

Large amounts of inactive parts indicate that background merges are behind. Check with:

```sql
SELECT *
FROM system.merges
ORDER BY elapsed DESC;
```

## Checking Temporary File Usage

ClickHouse writes temporary files during merges and external sorts:

```sql
SELECT
    formatReadableSize(sum(total_space)) AS tmp_space
FROM system.disks
WHERE path LIKE '%tmp%';
```

Also check the `tmp` subdirectory on each disk:

```bash
du -sh /var/lib/clickhouse/tmp/
```

## Checking Detached Parts

Detached parts are not included in `system.parts` but still consume disk space:

```sql
SELECT
    database,
    table,
    count()                                AS detached_parts,
    formatReadableSize(sum(bytes_on_disk)) AS size
FROM system.detached_parts
GROUP BY database, table
ORDER BY sum(bytes_on_disk) DESC;
```

To remove old detached parts (only after verifying they are safe to drop):

```sql
ALTER TABLE my_table DROP DETACHED PART 'part_name_here' SETTINGS allow_drop_detached = 1;
```

## Alerting on Low Free Space

Use this query as a health check - it returns rows only when a disk is above 85% full:

```sql
SELECT
    name                                           AS disk,
    formatReadableSize(free_space)                 AS free,
    round((1 - free_space / total_space) * 100, 2) AS used_pct
FROM system.disks
WHERE (1 - free_space / total_space) > 0.85;
```

Return code 0 means no disks are over threshold - useful for scripted checks:

```bash
clickhouse-client --query \
  "SELECT count() FROM system.disks WHERE (1 - free_space / total_space) > 0.85" \
  | grep -q '^0$' && echo "OK" || echo "ALERT: disk above 85%"
```

## Estimating Future Growth

Calculate daily ingestion rate and project disk exhaustion:

```sql
SELECT
    table,
    formatReadableSize(sum(bytes_on_disk))                       AS total_size,
    formatReadableSize(
        sum(bytes_on_disk) / dateDiff('day', min(modification_time), max(modification_time))
    )                                                             AS avg_daily_growth
FROM system.parts
WHERE active = 1
  AND database = currentDatabase()
GROUP BY table
HAVING count() > 1
ORDER BY sum(bytes_on_disk) DESC;
```

## Summary

ClickHouse provides complete disk visibility through `system.disks`, `system.parts`, and `system.detached_parts`. Query these tables regularly to track per-disk, per-table, and per-partition usage. Monitor inactive parts as a sign of merge lag. Build alerting queries that trigger when disks exceed a fill threshold, and project growth rates to plan capacity before running out of space.
