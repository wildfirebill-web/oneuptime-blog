# How to Use Null Table Engine in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Null Engine, Materialized View, Storage Engine, Pipeline

Description: Learn how to use the Null table engine in ClickHouse to discard data on write while triggering materialized views, enabling efficient pre-aggregation pipelines.

---

The `Null` table engine accepts all writes and immediately discards them. No data is stored, no disk space is used, and reads always return zero rows. At first glance this seems useless - but the Null engine is a powerful pipeline primitive because ClickHouse still fires materialized views attached to a Null table. This makes it the standard pattern for high-throughput pre-aggregation: raw data lands on a Null table, materialized views capture and aggregate it, and only the compact aggregated rows are persisted.

## Creating a Null Table

```sql
CREATE TABLE raw_events_null
(
    event_time  DateTime,
    user_id     UInt64,
    event_type  LowCardinality(String),
    page        String,
    duration_ms UInt32,
    country     LowCardinality(String)
)
ENGINE = Null;
```

## Confirming Null Behavior

Data inserted into a Null table vanishes immediately.

```sql
INSERT INTO raw_events_null VALUES
    (now(), 1001, 'page_view', '/home',    320, 'US'),
    (now(), 1002, 'click',     '/pricing', 0,   'DE');

-- Returns zero rows - data was discarded
SELECT count() FROM raw_events_null;
```

```text
count()
0
```

## Attaching a Materialized View

The real power of the Null engine is as a trigger for materialized views that write to real storage.

```sql
-- Target storage table - holds only the aggregated summaries
CREATE TABLE hourly_event_summary
(
    event_hour  DateTime,
    event_type  LowCardinality(String),
    country     LowCardinality(String),
    event_count UInt64,
    unique_users UInt64,
    avg_duration_ms Float64
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(event_hour)
ORDER BY (event_hour, event_type, country);

-- Materialized view reads from the Null table and writes to the summary
CREATE MATERIALIZED VIEW mv_hourly_events
TO hourly_event_summary
AS
SELECT
    toStartOfHour(event_time) AS event_hour,
    event_type,
    country,
    count()                   AS event_count,
    uniqExact(user_id)        AS unique_users,
    avg(duration_ms)          AS avg_duration_ms
FROM raw_events_null
GROUP BY event_hour, event_type, country;
```

Now inserting into the Null table triggers the materialized view and populates `hourly_event_summary`.

```sql
INSERT INTO raw_events_null VALUES
    ('2024-06-15 10:05:00', 1001, 'page_view', '/home',    320, 'US'),
    ('2024-06-15 10:12:00', 1002, 'click',     '/pricing',   0, 'DE'),
    ('2024-06-15 10:15:00', 1003, 'page_view', '/docs',    180, 'US'),
    ('2024-06-15 10:20:00', 1001, 'page_view', '/pricing', 410, 'US'),
    ('2024-06-15 10:30:00', 1004, 'sign_up',   '/register',  0, 'DE');

-- Raw rows were discarded - but the summary was populated
SELECT * FROM hourly_event_summary ORDER BY event_hour, event_type, country;
```

```text
event_hour            event_type  country  event_count  unique_users  avg_duration_ms
2024-06-15 10:00:00   click       DE       1            1             0
2024-06-15 10:00:00   page_view   US       3            2             303.33
2024-06-15 10:00:00   sign_up     DE       1            1             0
```

## Multiple Materialized Views on One Null Table

A single Null table can feed multiple independent materialized views, each writing to its own target.

```sql
-- Second summary: daily country rollup
CREATE TABLE daily_country_summary
(
    event_date   Date,
    country      LowCardinality(String),
    event_count  UInt64,
    unique_users UInt64
)
ENGINE = SummingMergeTree
ORDER BY (event_date, country);

CREATE MATERIALIZED VIEW mv_daily_country
TO daily_country_summary
AS
SELECT
    toDate(event_time)   AS event_date,
    country,
    count()              AS event_count,
    uniqExact(user_id)   AS unique_users
FROM raw_events_null
GROUP BY event_date, country;

-- Third summary: per-page performance
CREATE TABLE page_performance
(
    event_hour      DateTime,
    page            String,
    view_count      UInt64,
    avg_duration_ms Float64,
    p95_duration_ms Float64
)
ENGINE = MergeTree
ORDER BY (event_hour, page);

CREATE MATERIALIZED VIEW mv_page_perf
TO page_performance
AS
SELECT
    toStartOfHour(event_time) AS event_hour,
    page,
    count()                   AS view_count,
    avg(duration_ms)          AS avg_duration_ms,
    quantile(0.95)(duration_ms) AS p95_duration_ms
FROM raw_events_null
WHERE event_type = 'page_view'
GROUP BY event_hour, page;
```

A single `INSERT INTO raw_events_null` now populates all three summary tables simultaneously.

## Ingestion Pipeline Pattern

This is the canonical ClickHouse ingestion architecture for high-throughput event streams.

```sql
-- Step 1: insert the raw batch (Kafka consumer or application code calls this)
INSERT INTO raw_events_null
SELECT
    event_time,
    user_id,
    event_type,
    page,
    duration_ms,
    country
FROM input('event_time DateTime, user_id UInt64, event_type String,
            page String, duration_ms UInt32, country String')
FORMAT JSONEachRow;
```

```bash
# Feeding data from a file via clickhouse-client
clickhouse-client --query "INSERT INTO raw_events_null FORMAT JSONEachRow" \
  < events_batch.jsonl
```

## Listing Materialized Views Attached to a Null Table

```sql
SELECT
    name,
    engine,
    as_select
FROM system.tables
WHERE engine = 'MaterializedView'
  AND create_table_query LIKE '%raw_events_null%';
```

## Verifying No Storage Used

```sql
SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed,
    sum(rows)                                       AS row_count
FROM system.parts
WHERE database = currentDatabase()
  AND table IN ('raw_events_null', 'hourly_event_summary')
GROUP BY table;
```

```text
table                  compressed  row_count
hourly_event_summary   1.20 KiB    3
```

`raw_events_null` has no rows in `system.parts` because it stores nothing.

## Summary

The `Null` engine discards all inserted data but fires attached materialized views before doing so. This makes it the foundation of ClickHouse pre-aggregation pipelines: raw high-volume data passes through a Null table, materialized views capture only the aggregated summaries, and storage is consumed only by compact result tables. Use the Null engine whenever you want to decouple the ingestion schema from the storage schema.
