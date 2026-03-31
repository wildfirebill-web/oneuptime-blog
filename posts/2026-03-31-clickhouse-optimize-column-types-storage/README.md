# How to Optimize Column Types for Storage Efficiency in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Column Type, Storage, Compression, Performance, Schema Design

Description: Learn how to choose optimal ClickHouse column types and compression codecs to minimize storage while maintaining query performance for analytical workloads.

---

Column type selection in ClickHouse directly affects storage size, compression ratios, and query speed. Using the smallest type that fits your data range can reduce storage by 2-4x and improve query throughput by keeping more data in CPU cache.

## Integer Type Selection

Choose the smallest integer type that fits your value range:

```sql
-- Bad: using Int64 for everything
CREATE TABLE events_wasteful (
    event_id Int64,    -- UInt64 could work, or even UInt32
    user_id Int64,     -- If max users < 4B, UInt32 saves 4 bytes/row
    status_code Int64, -- 3-digit code: UInt16 saves 6 bytes/row
    age Int64          -- 0-150: UInt8 saves 7 bytes/row
);

-- Good: right-sized types
CREATE TABLE events_efficient (
    event_id UInt64,     -- 8 bytes, non-negative IDs
    user_id UInt32,      -- 4 bytes, up to 4.3B users
    status_code UInt16,  -- 2 bytes, 0-65535
    age UInt8            -- 1 byte, 0-255
);
```

## String vs. LowCardinality

`LowCardinality(String)` uses dictionary encoding, reducing storage for repeated values:

```sql
-- Check cardinality before choosing
SELECT uniq(event_type) AS unique_types FROM events;
-- If < 10,000 unique values: use LowCardinality

CREATE TABLE events (
    event_type LowCardinality(String),  -- few unique values
    country LowCardinality(String),      -- ~200 unique values
    session_id String,                   -- millions of unique values - keep as String
    user_agent String                    -- thousands - borderline
);
```

## Decimal vs. Float for Money

For financial data, use `Decimal` to avoid floating-point errors:

```sql
-- Risky: floating point precision issues
amount Float64

-- Safe: exact decimal arithmetic
amount Decimal(18, 2)  -- up to 16 digits before decimal, 2 after
price Decimal(10, 4)   -- 4 decimal places for FX rates
```

## Date and DateTime Types

Match temporal precision to actual needs:

```sql
-- Date: 4 bytes, day precision (sufficient for most event dates)
event_date Date

-- DateTime: 4 bytes, second precision (standard for events)
event_time DateTime

-- DateTime64: 8 bytes, microsecond precision (for tracing/logs)
event_time DateTime64(6)  -- 6 decimal places = microseconds

-- Don't use DateTime64 for date-level fields
-- Bad: event_date DateTime64(6) -- wastes bytes
```

## Compression Codecs

Add column-specific codecs for better compression:

```sql
CREATE TABLE events (
    -- Delta codec for monotonically increasing timestamps
    event_time DateTime CODEC(Delta, ZSTD(1)),

    -- Delta codec for sequential IDs
    event_id UInt64 CODEC(Delta, ZSTD(1)),

    -- ZSTD for general strings
    message String CODEC(ZSTD(3)),

    -- DoubleDelta for metrics with slow change
    cpu_usage Float32 CODEC(DoubleDelta, LZ4),

    -- Gorilla codec for floating-point time series
    temperature Float64 CODEC(Gorilla, ZSTD(1))
) ENGINE = MergeTree()
ORDER BY event_time;
```

## Nullable Columns

Avoid Nullable unless you genuinely have NULL values - each Nullable column stores an extra bitmap:

```sql
-- Avoid: Nullable adds storage and disables some optimizations
user_country Nullable(String)

-- Better: use empty string or sentinel value
user_country String DEFAULT ''

-- When you truly need nullable
user_country Nullable(LowCardinality(String))  -- if many repeated nulls
```

## Measuring Type Efficiency

Compare storage before and after type optimization:

```sql
SELECT
    name,
    type,
    formatReadableSize(data_compressed_bytes) AS compressed,
    formatReadableSize(data_uncompressed_bytes) AS uncompressed,
    round(data_uncompressed_bytes / data_compressed_bytes, 2) AS ratio
FROM system.columns
WHERE table = 'events' AND database = 'analytics'
ORDER BY data_compressed_bytes DESC;
```

## Quick Type Optimization Checklist

```sql
-- Find potentially oversized integer columns
SELECT name, type FROM system.columns
WHERE table = 'events'
  AND type IN ('Int64', 'UInt64')
  AND name NOT LIKE '%_id';  -- IDs legitimately need 64 bits

-- Find non-LowCardinality strings with low cardinality
SELECT name, type FROM system.columns
WHERE table = 'events'
  AND type = 'String';
-- Then check: SELECT uniq(col_name) FROM events LIMIT 1;
```

## Summary

Optimize ClickHouse column types by using the smallest integer type that fits your range, applying `LowCardinality` to strings with under 10K unique values, choosing `Decimal` for financial data, matching DateTime precision to actual needs, adding delta/Gorilla codecs for time-series columns, and avoiding Nullable unless genuinely necessary. These changes typically yield 2-4x storage reduction and 10-50% query speedup.
