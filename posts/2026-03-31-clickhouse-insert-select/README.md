# How to Use INSERT SELECT in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Insert, SELECT, Data Migration

Description: Learn how to use INSERT INTO ... SELECT in ClickHouse to copy, migrate, and transform data between tables efficiently.

---

`INSERT INTO ... SELECT` is one of the most powerful data movement tools in ClickHouse. It lets you read from one table (or any query) and write the results directly into another table - all within the server, without transferring data through a client. This makes it ideal for migrations, ETL pipelines, and creating derived datasets. This post walks through the syntax, common patterns, and performance considerations.

## Basic Syntax

The fundamental form mirrors standard SQL:

```sql
INSERT INTO target_table
SELECT col1, col2, col3
FROM source_table;
```

Both tables must have compatible column types for the selected columns. ClickHouse will coerce types where possible but will error on incompatible conversions.

## Copying an Entire Table

To duplicate all rows from one table to another with the same schema:

```sql
-- Create the destination table first
CREATE TABLE events_archive AS events;

-- Copy all data
INSERT INTO events_archive
SELECT *
FROM events;
```

Using `CREATE TABLE ... AS` copies the schema without data, giving you an identical structure to insert into.

## Transforming Data During Insert

`INSERT SELECT` is not limited to simple copies. You can apply any SELECT expression - filtering, casting, computing new columns:

```sql
INSERT INTO events_normalized
    (event_id, event_name, created_date, hour_of_day)
SELECT
    event_id,
    lower(event_name)              AS event_name,
    toDate(created_at)             AS created_date,
    toHour(created_at)             AS hour_of_day
FROM events_raw
WHERE created_at >= '2026-01-01 00:00:00'
  AND event_name != '';
```

Type casting during insert:

```sql
INSERT INTO metrics_typed
    (metric_id, value, recorded_at)
SELECT
    toUInt64(id),
    toFloat64(raw_value),
    parseDateTimeBestEffort(ts_string)
FROM metrics_staging;
```

## Partition Filtering

When copying large tables, filtering by partition key reduces the amount of data scanned and inserted. Use `_partition_id` or partition-key columns directly:

```sql
-- Copy only data from March 2026
INSERT INTO events_archive
SELECT *
FROM events
WHERE toYYYYMM(created_at) = 202603;
```

You can also leverage the `PARTITION` clause on the source with `ALTER TABLE ... MOVE PARTITION` for non-destructive moves, but `INSERT SELECT` with a partition filter is simpler when transformation is involved.

## Inserting from Multiple Sources with UNION ALL

Combine data from several tables in one pass:

```sql
INSERT INTO events_combined
SELECT event_id, event_name, created_at, 'web'    AS source FROM events_web
UNION ALL
SELECT event_id, event_name, created_at, 'mobile' AS source FROM events_mobile
UNION ALL
SELECT event_id, event_name, created_at, 'api'    AS source FROM events_api;
```

## Incremental Loads

For incremental pipelines, filter on a watermark column:

```sql
-- Load only new rows since the last run
INSERT INTO events_dw
SELECT *
FROM events_staging
WHERE created_at > (
    SELECT max(created_at) FROM events_dw
);
```

This pattern works well for append-only tables. For tables with updates, consider using `ReplacingMergeTree` as the destination and relying on deduplication at query time.

## Checking Progress for Large Inserts

For large `INSERT SELECT` operations, monitor progress in `system.processes`:

```sql
SELECT
    query_id,
    elapsed,
    read_rows,
    written_rows,
    memory_usage
FROM system.processes
WHERE query ILIKE 'INSERT INTO%'
ORDER BY elapsed DESC;
```

## Performance Tips

- Avoid running many concurrent `INSERT SELECT` operations on the same destination table; they serialize at the part level and can cause merge pressure.
- Add `SETTINGS max_insert_block_size = 1048576` to control block size during large inserts.
- For very large migrations, consider splitting by partition and running each partition sequentially.

```sql
-- Controlled large insert with explicit block size
INSERT INTO events_archive
SELECT *
FROM events
SETTINGS max_insert_block_size = 1048576;
```

## Summary

`INSERT INTO ... SELECT` in ClickHouse is a server-side, zero-copy-to-client operation that supports full SQL transformations, partition filtering, and multi-source unions. It is the go-to tool for table migrations, ETL pipelines, and incremental loads. Keep batches manageable and monitor `system.processes` for long-running inserts.
