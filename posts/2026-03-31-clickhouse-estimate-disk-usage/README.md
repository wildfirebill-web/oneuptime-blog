# How to Estimate Future Disk Usage in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Capacity Planning, Disk Usage, Storage, Monitoring

Description: Learn how to estimate future ClickHouse disk usage by analyzing current growth rates, compression ratios, and data retention policies for capacity planning.

---

Running out of disk space in production is avoidable with proper capacity planning. By analyzing current data growth rates and compression characteristics, you can accurately forecast future storage needs and provision accordingly.

## Assessing Current Storage Usage

Start with a full picture of current usage:

```sql
SELECT
    database,
    table,
    formatReadableSize(sum(bytes_on_disk)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(bytes_on_disk), 2) AS compression_ratio,
    sum(rows) AS total_rows,
    count() AS parts
FROM system.parts
WHERE active = 1
  AND database NOT IN ('system', 'information_schema')
GROUP BY database, table
ORDER BY sum(bytes_on_disk) DESC;
```

## Calculating Daily Data Growth Rate

Analyze how much data is added each day:

```sql
-- Daily ingestion volume for the last 30 days
SELECT
    toDate(modification_time) AS day,
    formatReadableSize(sum(bytes_on_disk)) AS daily_size,
    sum(rows) AS daily_rows
FROM system.parts
WHERE active = 1
  AND database = 'analytics'
  AND table = 'events'
  AND modification_time >= today() - 30
GROUP BY day
ORDER BY day;
```

Average daily growth:

```sql
SELECT
    formatReadableSize(avg(daily_bytes)) AS avg_daily_growth,
    formatReadableSize(max(daily_bytes)) AS peak_daily_growth
FROM (
    SELECT
        toDate(modification_time) AS day,
        sum(bytes_on_disk) AS daily_bytes
    FROM system.parts
    WHERE active = 1
      AND database = 'analytics'
      AND table = 'events'
      AND modification_time >= today() - 30
    GROUP BY day
);
```

## Projecting Future Storage Requirements

Using the average daily growth rate to project storage needs:

```sql
WITH
    avg_daily AS (
        SELECT avg(daily_bytes) AS rate
        FROM (
            SELECT toDate(modification_time) AS day, sum(bytes_on_disk) AS daily_bytes
            FROM system.parts
            WHERE active = 1 AND database = 'analytics' AND table = 'events'
              AND modification_time >= today() - 30
            GROUP BY day
        )
    ),
    current AS (
        SELECT sum(bytes_on_disk) AS total
        FROM system.parts
        WHERE active = 1 AND database = 'analytics' AND table = 'events'
    )
SELECT
    formatReadableSize(current.total) AS current_size,
    formatReadableSize(current.total + avg_daily.rate * 30) AS in_30_days,
    formatReadableSize(current.total + avg_daily.rate * 90) AS in_90_days,
    formatReadableSize(current.total + avg_daily.rate * 365) AS in_1_year
FROM current, avg_daily;
```

## Factoring in Compression

Estimate raw data volume before compression:

```sql
SELECT
    database,
    table,
    round(sum(data_uncompressed_bytes) / sum(bytes_on_disk), 2) AS compression_ratio
FROM system.parts
WHERE active = 1
GROUP BY database, table
ORDER BY compression_ratio DESC;
```

Higher compression ratio = more efficient storage. ClickHouse typically achieves 5-15x compression for time-series data.

## Accounting for TTL Data Deletion

If you have TTL policies, calculate net growth:

```sql
-- Estimate data deleted by TTL each day
SELECT
    toDate(event_time) AS day,
    count() AS rows_to_expire
FROM analytics.events
WHERE event_time < today() - 90  -- TTL threshold
GROUP BY day
ORDER BY day DESC
LIMIT 30;
```

Net growth = Daily ingest - Daily TTL expiry

## Monitoring Disk Usage Over Time

Create a baseline monitoring query:

```bash
#!/bin/bash
# Log disk usage metrics daily
clickhouse-client --query "
SELECT
    now() AS ts,
    name AS disk,
    total_space,
    free_space,
    total_space - free_space AS used_space
FROM system.disks
FORMAT CSV
" >> /var/log/clickhouse-disk-usage.csv
```

## Summary

Estimating future ClickHouse disk usage requires measuring current storage size, calculating daily growth rates from `system.parts`, and projecting forward while accounting for TTL deletions and compression ratios. Run this analysis monthly and compare against actual growth to maintain accurate capacity forecasts. Alert when projected usage will exceed capacity within 30 days.
