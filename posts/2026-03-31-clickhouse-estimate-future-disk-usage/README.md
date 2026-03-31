# How to Estimate Future Disk Usage in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Disk Space, Capacity Planning, system.part_log, Monitoring

Description: Learn how to analyze ClickHouse growth trends and project future disk usage to plan capacity before running out of space.

---

Capacity planning for ClickHouse requires understanding how fast data is growing, how compression affects storage, and when TTL policies will start expiring data. This post shows how to build accurate disk usage projections.

## Current Disk Usage Baseline

Start with a clear picture of current state:

```sql
SELECT
    database,
    table,
    formatReadableSize(sum(bytes_on_disk)) AS disk_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS compression_ratio,
    sum(rows) AS total_rows
FROM system.parts
WHERE active = 1
GROUP BY database, table
ORDER BY sum(bytes_on_disk) DESC;
```

## Measuring Daily Growth Rate

Calculate average daily data ingestion from `system.part_log`:

```sql
SELECT
    database,
    table,
    toDate(event_time) AS day,
    formatReadableSize(sum(bytes_on_disk)) AS bytes_added,
    sum(rows) AS rows_added
FROM system.part_log
WHERE
    event_type = 'NewPart'
    AND event_time >= now() - INTERVAL 30 DAY
GROUP BY database, table, day
ORDER BY database, table, day;
```

## Calculating Average Daily Growth

```sql
SELECT
    database,
    table,
    formatReadableSize(avg(daily_bytes)) AS avg_daily_growth,
    formatReadableSize(avg(daily_bytes) * 30) AS est_monthly_growth,
    formatReadableSize(avg(daily_bytes) * 365) AS est_annual_growth
FROM (
    SELECT
        database,
        table,
        toDate(event_time) AS day,
        sum(bytes_on_disk) AS daily_bytes
    FROM system.part_log
    WHERE
        event_type = 'NewPart'
        AND event_time >= now() - INTERVAL 30 DAY
    GROUP BY database, table, day
)
GROUP BY database, table
ORDER BY avg(daily_bytes) DESC;
```

## Projecting Days Until Disk Full

Combine growth rate with available disk space:

```sql
WITH
growth_rates AS (
    SELECT
        table,
        avg(daily_bytes) AS avg_daily_bytes
    FROM (
        SELECT
            table,
            toDate(event_time) AS day,
            sum(bytes_on_disk) AS daily_bytes
        FROM system.part_log
        WHERE event_type = 'NewPart'
          AND event_time >= now() - INTERVAL 30 DAY
        GROUP BY table, day
    )
    GROUP BY table
),
disk_free AS (
    SELECT free_space
    FROM system.disks
    WHERE name = 'default'
)
SELECT
    table,
    formatReadableSize(avg_daily_bytes) AS daily_growth,
    formatReadableSize(disk_free.free_space) AS disk_free,
    round(disk_free.free_space / avg_daily_bytes) AS days_until_full,
    today() + INTERVAL round(disk_free.free_space / avg_daily_bytes) DAY AS projected_full_date
FROM growth_rates, disk_free
ORDER BY days_until_full;
```

## Accounting for TTL Expiry

If you have TTL DELETE policies, factor in data expiry:

```sql
SELECT
    table,
    partition,
    sum(rows) AS rows,
    formatReadableSize(sum(bytes_on_disk)) AS size,
    min(min_time) AS oldest_data
FROM system.parts
WHERE active = 1
  AND database = 'production'
GROUP BY table, partition
HAVING oldest_data < now() - INTERVAL 90 DAY  -- TTL threshold
ORDER BY table, partition;
```

These partitions will be deleted by TTL, reducing net growth.

## Estimating Compression Ratio for New Data

Use historical compression ratios to project compressed storage from raw data volume:

```sql
SELECT
    table,
    avg(bytes_on_disk / data_uncompressed_bytes) AS avg_compression_ratio,
    avg(bytes_on_disk) / avg(data_uncompressed_bytes) AS compressed_fraction
FROM system.parts
WHERE active = 1
GROUP BY table;
```

If your application generates 1TB/day of raw data and average `compressed_fraction` is 0.12, you need 120GB/day of storage.

## Building a Capacity Plan

Combine all factors:

```bash
# Summarize key metrics to a report
clickhouse-client --query "
SELECT
    'Current disk usage' AS metric,
    formatReadableSize(sum(bytes_on_disk)) AS value
FROM system.parts WHERE active = 1
UNION ALL
SELECT
    'Available disk space',
    formatReadableSize(free_space)
FROM system.disks WHERE name = 'default'
FORMAT TSVWithNames"
```

## Summary

ClickHouse disk usage estimation combines current usage from `system.parts`, daily growth trends from `system.part_log`, and TTL expiry projections to forecast when disks will fill. Calculate `days_until_full` as a key metric and alert when it drops below 30 days. Factor in compression ratios when converting application data volume to ClickHouse storage requirements.
