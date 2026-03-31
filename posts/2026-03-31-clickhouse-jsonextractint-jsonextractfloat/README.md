# How to Use JSONExtractInt() and JSONExtractFloat() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Json, Analytics, Query

Description: Learn how JSONExtractInt() and JSONExtractFloat() pull numeric values from JSON strings stored in ClickHouse columns, with real query examples and type handling tips.

---

ClickHouse stores semi-structured data as plain JSON strings in `String` columns. `JSONExtractInt` and `JSONExtractFloat` let you reach into those strings at query time to extract typed numeric values without pre-parsing or schema changes. `JSONExtractInt` returns an `Int64` and `JSONExtractFloat` returns a `Float64`. Both return `0` when the path is missing or the value cannot be cast, so it is worth understanding that default before using them in aggregations.

## Basic Extraction

```sql
-- Extract integer and float fields from a JSON string literal
SELECT
    JSONExtractInt('{"user_id": 42, "score": 98.7}', 'user_id')   AS user_id,
    JSONExtractFloat('{"user_id": 42, "score": 98.7}', 'score')   AS score;
```

```text
user_id  score
42       98.7
```

The first argument is the JSON string (or a column name), and the second is the top-level key to extract.

## Extracting from a Table Column

```sql
-- events table has a String column called payload
SELECT
    event_id,
    JSONExtractInt(payload, 'duration_ms')    AS duration_ms,
    JSONExtractFloat(payload, 'cpu_percent')  AS cpu_pct
FROM events
LIMIT 5;
```

## Filtering on Extracted Numbers

Extracted values can appear directly in `WHERE` clauses, though ClickHouse cannot use primary index scans on virtual computed columns.

```sql
-- Find events where duration exceeded 500 ms
SELECT
    event_id,
    JSONExtractInt(payload, 'duration_ms') AS duration_ms
FROM events
WHERE JSONExtractInt(payload, 'duration_ms') > 500
ORDER BY duration_ms DESC
LIMIT 20;
```

## Aggregating Extracted Numbers

```sql
-- Average CPU usage and max memory per service extracted from JSON payloads
SELECT
    JSONExtractString(payload, 'service')     AS service,
    avg(JSONExtractFloat(payload, 'cpu_pct')) AS avg_cpu,
    max(JSONExtractInt(payload, 'mem_mb'))    AS max_mem_mb,
    count()                                   AS samples
FROM metrics
GROUP BY service
ORDER BY avg_cpu DESC;
```

## Handling Missing Keys

When the key does not exist, both functions return `0`. Use a conditional to distinguish zero from missing.

```sql
-- Differentiate between a genuine 0 and a missing field
SELECT
    event_id,
    JSONHas(payload, 'retry_count')             AS has_retry_count,
    JSONExtractInt(payload, 'retry_count')       AS retry_count
FROM events
LIMIT 10;
```

## Nested Key Extraction

For nested JSON, pass the path as successive arguments after the JSON string.

```sql
-- Extract a nested integer: {"meta": {"priority": 3}}
SELECT
    JSONExtractInt(payload, 'meta', 'priority')        AS priority,
    JSONExtractFloat(payload, 'metrics', 'p99_ms')     AS p99_ms
FROM requests
LIMIT 10;
```

## Converting Extracted Values to Other Types

If you need a smaller integer type or a `Decimal`, wrap the result with an explicit cast.

```sql
-- Cast to UInt32 and Decimal for downstream use
SELECT
    toUInt32(JSONExtractInt(payload, 'status_code'))        AS status_code,
    toDecimal64(JSONExtractFloat(payload, 'amount'), 2)     AS amount
FROM transactions
LIMIT 5;
```

## Materialized Column Pattern

Repeatedly calling `JSONExtractInt` on every query is expensive on large tables. A materialized column computes the value once at insert time.

```sql
ALTER TABLE events
    ADD COLUMN duration_ms Int64
    MATERIALIZED JSONExtractInt(payload, 'duration_ms');
```

After back-filling, queries can filter on `duration_ms` directly and benefit from index scans.

## Summary

`JSONExtractInt` and `JSONExtractFloat` are the primary tools for reading numeric data from JSON strings in ClickHouse. They handle nested paths via variadic arguments and return `0` for missing keys, so use `JSONHas` when you need to distinguish absence from a genuine zero. For hot columns that are filtered or aggregated frequently, promote them to materialized columns to avoid repeated parsing overhead.
