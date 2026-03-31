# How to Alter Materialized Views in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, DDL, Materialized View, ALTER

Description: Learn how to modify materialized views in ClickHouse, work around ALTER limitations, modify target tables, and recreate views safely with OR REPLACE.

---

Materialized views in ClickHouse are powerful real-time aggregation pipelines that trigger on INSERT and write results to a target table. However, modifying an existing materialized view is more restricted than altering a regular table. Understanding what can and cannot be changed - and the patterns for working around those limits - is essential for maintaining production pipelines without data loss.

## How Materialized Views Work

A materialized view consists of two objects:
- A view definition (the SELECT query that transforms incoming data).
- A target table where results are written (either explicit or auto-created).

When rows are inserted into the source table, ClickHouse executes the view's SELECT against the new block of data and inserts the result into the target table.

## What You Can and Cannot Alter

ClickHouse does not support modifying the SELECT query of an existing materialized view with a simple `ALTER` statement. You cannot change the transformation logic, add or remove columns from the view's query, or change the source table.

What you **can** do:
- Alter the **target table** using standard `ALTER TABLE` statements.
- Replace the entire view using `CREATE OR REPLACE MATERIALIZED VIEW`.
- Drop and recreate the view.

## Modifying the Target Table

If you only need to add a column to the output, alter the target table directly:

```sql
-- Materialized view writes to: mv_daily_stats
ALTER TABLE mv_daily_stats
    ADD COLUMN p95_duration_ms Float64 DEFAULT 0;
```

The view will continue writing to the target table. New rows will include the new column if the view SELECT produces it; otherwise the column receives its default value.

To add the column to the view's output as well, you must recreate the view (see below).

## Recreating a View with OR REPLACE

`CREATE OR REPLACE MATERIALIZED VIEW` atomically replaces an existing view definition without dropping intermediate objects:

```sql
CREATE OR REPLACE MATERIALIZED VIEW mv_daily_stats
TO daily_stats_table
AS
SELECT
    toDate(event_time)          AS event_date,
    event_type,
    count()                     AS total_events,
    avg(duration_ms)            AS avg_duration_ms,
    quantile(0.95)(duration_ms) AS p95_duration_ms
FROM events
GROUP BY event_date, event_type;
```

This replaces the SELECT logic while keeping the target table (`daily_stats_table`) and its existing data intact. New inserts after the replacement will use the updated query.

## Changing the Target Table Schema First

When adding computed columns, update the target table before replacing the view to avoid schema mismatches:

```sql
-- Step 1: add column to target table
ALTER TABLE daily_stats_table
    ADD COLUMN p95_duration_ms Float64 DEFAULT 0;

-- Step 2: replace view to include new column in SELECT
CREATE OR REPLACE MATERIALIZED VIEW mv_daily_stats
TO daily_stats_table
AS
SELECT
    toDate(event_time)          AS event_date,
    event_type,
    count()                     AS total_events,
    avg(duration_ms)            AS avg_duration_ms,
    quantile(0.95)(duration_ms) AS p95_duration_ms
FROM events
GROUP BY event_date, event_type;
```

## Backfilling Historical Data

Replacing a view does not backfill historical data - only new inserts are processed by the updated query. To backfill, insert from the source table into the target table manually:

```sql
INSERT INTO daily_stats_table
SELECT
    toDate(event_time)          AS event_date,
    event_type,
    count()                     AS total_events,
    avg(duration_ms)            AS avg_duration_ms,
    quantile(0.95)(duration_ms) AS p95_duration_ms
FROM events
WHERE event_time >= '2024-01-01' AND event_time < '2024-06-01'
GROUP BY event_date, event_type;
```

Be aware that this may produce duplicate aggregated rows if the target table uses a SummingMergeTree or AggregatingMergeTree engine - those engines handle deduplication on merge.

## Inspecting the Current View Definition

Before modifying a view, inspect its current definition:

```sql
SELECT
    name,
    create_table_query
FROM system.tables
WHERE engine = 'MaterializedView'
  AND database = 'default';
```

Or use:

```sql
SHOW CREATE MATERIALIZED VIEW mv_daily_stats;
```

## Handling View Dependencies

If multiple materialized views read from the same source table, altering one does not affect others. Track all views on a source table:

```sql
SELECT
    name,
    engine,
    as_select
FROM system.tables
WHERE engine = 'MaterializedView'
  AND database = 'default'
ORDER BY name;
```

## Summary

ClickHouse does not support directly altering the SELECT logic of a materialized view. The recommended pattern is to alter the target table for schema changes and use `CREATE OR REPLACE MATERIALIZED VIEW` to update the transformation query. Historical data is never automatically backfilled - run a manual INSERT SELECT to populate past data after a view change.
