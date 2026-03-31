# How to Use LowCardinality for Better Query Performance in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, LowCardinality, Performance, Storage Optimization, Data Types

Description: Learn how to use the LowCardinality type modifier in ClickHouse to speed up queries and reduce storage for string columns with few distinct values.

---

## What Is LowCardinality in ClickHouse

`LowCardinality` is a data type modifier that wraps any type (most commonly `String`) and stores values using dictionary encoding. Instead of storing the full string for every row, ClickHouse stores a compact numeric ID per row and a separate dictionary of unique values.

This works well for columns with low cardinality - fewer than roughly 10,000 distinct values.

## Creating a Table with LowCardinality

```sql
CREATE TABLE events (
    ts          DateTime,
    user_id     UInt64,
    event_type  LowCardinality(String),   -- e.g. 'click', 'view', 'purchase'
    country     LowCardinality(String),   -- e.g. 'US', 'UK', 'DE'
    platform    LowCardinality(String),   -- e.g. 'web', 'ios', 'android'
    status      LowCardinality(String)    -- e.g. 'success', 'error', 'timeout'
) ENGINE = MergeTree()
ORDER BY (ts, user_id);
```

## How LowCardinality Improves Performance

1. **Compression**: Dictionary encoding compresses repeated strings to small integers
2. **Faster GROUP BY**: Aggregations use integer IDs internally
3. **Faster ORDER BY**: Sorting integers is faster than sorting strings
4. **Reduced I/O**: Smaller on-disk footprint means less data to read

## Adding LowCardinality to an Existing Column

```sql
ALTER TABLE events
    MODIFY COLUMN event_type LowCardinality(String);

-- Verify
DESCRIBE TABLE events;
```

## Measuring the Impact

```sql
-- Create two test tables
CREATE TABLE test_regular (
    event_type String
) ENGINE = MergeTree() ORDER BY tuple();

CREATE TABLE test_low_card (
    event_type LowCardinality(String)
) ENGINE = MergeTree() ORDER BY tuple();

-- Insert same data
INSERT INTO test_regular
    SELECT arrayElement(['click','view','purchase','login','logout'], rand() % 5 + 1)
    FROM numbers(10000000);

INSERT INTO test_low_card
    SELECT arrayElement(['click','view','purchase','login','logout'], rand() % 5 + 1)
    FROM numbers(10000000);

-- Compare sizes
SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed
FROM system.parts
WHERE database = 'default'
  AND table IN ('test_regular', 'test_low_card')
  AND active = 1
GROUP BY table;
```

## LowCardinality with Other Types

LowCardinality works with numeric types too:

```sql
CREATE TABLE metrics (
    ts          DateTime,
    host_id     LowCardinality(UInt32),   -- Small set of server IDs
    metric_name LowCardinality(String),
    value       Float64
) ENGINE = MergeTree()
ORDER BY (metric_name, ts);
```

## When NOT to Use LowCardinality

Avoid `LowCardinality` for high-cardinality columns:

```sql
-- BAD: user_id has millions of distinct values
user_id LowCardinality(UInt64)  -- Don't do this

-- BAD: UUID or random string per row
session_id LowCardinality(String)  -- Don't do this

-- GOOD: limited set of values
event_type LowCardinality(String)  -- Good for ~5-100 distinct values
```

If there are more than ~10,000 distinct values, the dictionary becomes large and performance degrades.

## Checking Cardinality Before Applying

```sql
-- Count distinct values to decide if LowCardinality is appropriate
SELECT
    uniq(event_type) AS event_types,
    uniq(country) AS countries,
    uniq(user_id) AS users
FROM events;
```

## LowCardinality with Nullable

```sql
CREATE TABLE events (
    event_type LowCardinality(Nullable(String))
) ENGINE = MergeTree() ORDER BY tuple();
```

Note: `LowCardinality(Nullable(String))` is supported but slightly less efficient than `LowCardinality(String)`.

## LowCardinality in Distributed Tables

LowCardinality works seamlessly in distributed and replicated tables:

```sql
CREATE TABLE events ON CLUSTER my_cluster (
    ts          DateTime,
    event_type  LowCardinality(String),
    country     LowCardinality(String)
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
ORDER BY (ts);
```

## Summary

`LowCardinality(String)` is one of the most impactful quick wins in ClickHouse for string columns with fewer than ~10,000 distinct values. It reduces storage by 4-10x for repeated strings, speeds up GROUP BY and ORDER BY by operating on integer IDs, and has minimal query API changes since it's transparent to SQL. Check distinct value counts with `uniq()` before applying, and avoid it for high-cardinality columns like user IDs or session tokens.
