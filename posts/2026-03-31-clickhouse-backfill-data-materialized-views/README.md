# How to Backfill Data in Materialized Views in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Materialized View, Backfill, Data Migration, Analytics

Description: Learn how to backfill historical data into ClickHouse materialized view targets using INSERT SELECT, POPULATE, and manual partition strategies.

---

## Why Backfilling is Needed

Materialized views in ClickHouse only process data inserted after the view is created. If you have existing historical data in the source table, you need to manually backfill the target table.

## Method 1: INSERT SELECT

The most reliable approach - run the view's SELECT query manually and insert results into the target:

```sql
-- Assumes mv_hourly writes to events_hourly
INSERT INTO events_hourly
SELECT
    toStartOfHour(event_time) AS hour,
    event_type,
    count() AS cnt,
    sum(value) AS total_value
FROM raw_events
WHERE event_time < now()
GROUP BY hour, event_type;
```

## Method 2: POPULATE Keyword

When creating a new view, add `POPULATE` to backfill immediately. Note: new inserts during population may be lost.

```sql
CREATE MATERIALIZED VIEW mv_events_hourly
TO events_hourly
POPULATE
AS SELECT
    toStartOfHour(event_time) AS hour,
    event_type,
    count() AS cnt
FROM raw_events
GROUP BY hour, event_type;
```

Warning: `POPULATE` is not safe for production tables with active writes. Use INSERT SELECT instead.

## Method 3: Backfill by Month Partition

For large tables, backfill month by month to avoid memory pressure:

```sql
-- Backfill January 2025
INSERT INTO events_hourly
SELECT
    toStartOfHour(event_time) AS hour,
    event_type,
    count() AS cnt
FROM raw_events
WHERE toYYYYMM(event_time) = 202501
GROUP BY hour, event_type;

-- Backfill February 2025
INSERT INTO events_hourly
SELECT
    toStartOfHour(event_time) AS hour,
    event_type,
    count() AS cnt
FROM raw_events
WHERE toYYYYMM(event_time) = 202502
GROUP BY hour, event_type;
```

## Automate with a Shell Script

```bash
#!/bin/bash
START_MONTH=202501
END_MONTH=202512

for month in $(seq $START_MONTH $END_MONTH); do
    echo "Backfilling month $month..."
    clickhouse-client --query "
        INSERT INTO events_hourly
        SELECT toStartOfHour(event_time), event_type, count()
        FROM raw_events
        WHERE toYYYYMM(event_time) = $month
        GROUP BY 1, 2
    "
done
```

## Verify Backfill Completeness

```sql
-- Compare row counts between source and target
SELECT
    toYYYYMM(event_time) AS month,
    count() AS raw_count
FROM raw_events
GROUP BY month
ORDER BY month;

SELECT
    toYYYYMM(hour) AS month,
    sum(cnt) AS aggregated_count
FROM events_hourly
GROUP BY month
ORDER BY month;
```

## Handle Duplicates After Backfill

If you used `POPULATE` and some data got double-counted, use OPTIMIZE to deduplicate on ReplacingMergeTree targets:

```sql
OPTIMIZE TABLE events_hourly FINAL;
```

## Summary

Backfilling ClickHouse materialized view targets requires manually running the view's SELECT logic against historical data. The safest approach is using INSERT SELECT with partition-by-partition processing. Avoid POPULATE on production tables with active writes, and always verify completeness after backfilling.
