# How to Optimize JSON Parsing Performance in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, JSON, Performance, JSONExtract, Schema, Query Optimization

Description: Optimize JSON parsing performance in ClickHouse by pre-extracting columns at insert time, using proper JSON functions, and avoiding repeated parsing.

---

Parsing JSON at query time in ClickHouse is expensive because it happens row-by-row. The best approach is to parse JSON once at insert time and store columns natively.

## The Problem with Runtime JSON Parsing

Extracting fields from a JSON string column at query time parses the entire JSON for every row:

```sql
-- Slow: parses JSON for every row on every query
SELECT
  JSONExtractString(payload, 'user_id') AS user_id,
  JSONExtractFloat(payload, 'amount') AS amount
FROM events
WHERE JSONExtractString(payload, 'region') = 'US';
```

## Extract Columns at Insert Time

Use materialized columns to parse JSON once when data is inserted:

```sql
CREATE TABLE events (
  raw_payload String,
  -- Materialized columns parsed at insert time
  user_id String MATERIALIZED JSONExtractString(raw_payload, 'user_id'),
  amount Float64 MATERIALIZED JSONExtractFloat(raw_payload, 'amount'),
  region LowCardinality(String) MATERIALIZED JSONExtractString(raw_payload, 'region'),
  event_time DateTime MATERIALIZED parseDateTimeBestEffort(
    JSONExtractString(raw_payload, 'timestamp')
  )
) ENGINE = MergeTree()
ORDER BY (region, event_time);
```

Now queries use the materialized columns directly:

```sql
SELECT region, sum(amount)
FROM events
WHERE region = 'US'
GROUP BY region;
-- No JSON parsing at query time
```

## Use the JSON Data Type (Experimental)

ClickHouse 24+ has a native JSON type that automatically extracts paths:

```sql
SET allow_experimental_json_type = 1;

CREATE TABLE events_json (
  id UInt64,
  data JSON
) ENGINE = MergeTree()
ORDER BY id;

INSERT INTO events_json VALUES (1, '{"user": "alice", "amount": 42.5}');

SELECT data.user, data.amount FROM events_json;
```

## Choose the Right JSON Function

Different JSON functions have different performance characteristics:

```sql
-- For simple string extraction (fastest)
SELECT JSONExtractString(payload, 'event_type') FROM events;

-- For nested paths
SELECT JSONExtractString(payload, 'user', 'profile', 'name') FROM events;

-- For raw value (avoids type conversion)
SELECT JSONExtractRaw(payload, 'metadata') FROM events;

-- For checking key existence
SELECT JSONHas(payload, 'error_code') FROM events;
```

## Avoid JSON in WHERE Clauses

If you must parse JSON at query time, add a skip index or pre-filter to reduce rows processed:

```sql
-- Pre-filter with partitioning, then parse
SELECT JSONExtractString(payload, 'user_id')
FROM events
WHERE toDate(event_time) = today()  -- partition pruning first
  AND JSONExtractString(payload, 'type') = 'error';
```

## Cache JSON Extraction in a Materialized View

For frequently accessed JSON fields, create a materialized view:

```sql
CREATE MATERIALIZED VIEW events_parsed
ENGINE = MergeTree()
ORDER BY (region, event_time)
AS
SELECT
  JSONExtractString(raw, 'region') AS region,
  JSONExtractFloat(raw, 'amount') AS amount,
  parseDateTimeBestEffort(JSONExtractString(raw, 'ts')) AS event_time
FROM raw_events;
```

## Summary

JSON parsing performance in ClickHouse is best when JSON is parsed once at insert time using materialized columns or materialized views. For existing schemas, use partition pruning and skip indexes to minimize rows that need JSON parsing. The experimental JSON data type in ClickHouse 24+ offers automatic sub-column extraction without schema changes.
