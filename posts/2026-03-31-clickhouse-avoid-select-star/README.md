# Why You Should Avoid SELECT * in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Query Optimization, Performance, Best Practice, Columnar Storage

Description: Explains why SELECT * is especially costly in ClickHouse's columnar storage model and how to fix queries to read only the columns you need.

---

## The Columnar Storage Cost

In row-oriented databases like PostgreSQL, a row is stored together on disk. Reading one column or all columns costs nearly the same per row. ClickHouse stores each column in separate files. When you use `SELECT *`, ClickHouse must read every column file from disk - even the ones your query does not use.

On a table with 50 columns, a `SELECT *` that only needed 3 columns reads 16x more data than necessary.

## Measuring the Impact

```sql
-- Bad: reads all columns
SELECT *
FROM page_views
WHERE date = today()
LIMIT 10;

-- Good: reads only what you need
SELECT user_id, page_url, event_time
FROM page_views
WHERE date = today()
LIMIT 10;
```

Check actual bytes read with:

```sql
SELECT
  read_bytes,
  read_rows,
  query
FROM system.query_log
WHERE query LIKE '%page_views%'
  AND type = 'QueryFinish'
ORDER BY event_time DESC
LIMIT 5;
```

## Compression Amplification

ClickHouse compresses each column independently. Columnar compression is very effective because values in the same column tend to be similar. When you read columns you do not need, you decompress data that is immediately discarded - wasting CPU cycles and memory bandwidth.

## Schema Evolution Risk

`SELECT *` returns columns in schema-defined order. If someone adds or reorders columns, downstream consumers that destructure the result by position silently break.

```sql
-- Fragile: relies on column order
INSERT INTO summary SELECT * FROM source_table;

-- Safe: explicit column list
INSERT INTO summary (user_id, amount, created_at)
SELECT user_id, amount, created_at FROM source_table;
```

## Materialized Views

Materialized views in ClickHouse trigger on `INSERT`. A view defined as `SELECT *` will capture all source columns. As the source table grows with new columns, the materialized view silently starts storing more data than intended.

```sql
-- Avoid in materialized view definitions
CREATE MATERIALIZED VIEW mv_summary TO summary AS
SELECT * FROM raw_events;  -- dangerous

-- Prefer explicit columns
CREATE MATERIALIZED VIEW mv_summary TO summary AS
SELECT user_id, event_type, toStartOfHour(event_time) AS hour
FROM raw_events;
```

## When SELECT * Is Acceptable

- Interactive exploration in the ClickHouse client on small datasets
- Schema inspection queries like `SELECT * FROM system.tables LIMIT 1`
- One-off debugging with `LIMIT 1` to see what a row looks like

## Summary

`SELECT *` in ClickHouse reads every column file from disk, defeats compression benefits for unused columns, and creates fragile dependencies on schema order. Always list the columns your query needs. For wide tables with 20 or more columns, explicit column selection can reduce query time and bytes read by an order of magnitude.
