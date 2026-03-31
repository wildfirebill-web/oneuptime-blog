# How to Optimize ClickHouse Schema for Analytical Queries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Performance, SQL, Analytics, Database

Description: Learn how to design ClickHouse schemas optimized for analytical queries by choosing the right data types, primary keys, partitioning strategies, and compression codecs.

## Introduction

Schema design is the foundation of ClickHouse performance. Unlike relational databases where you can add indexes after the fact without dramatically changing query plans, ClickHouse query performance is primarily determined at schema design time: the `ORDER BY` key controls the sparse index, `PARTITION BY` controls partition elimination, and column data types determine compression ratios and vectorized execution efficiency. Getting these right before data ingestion starts is critical.

This guide covers a systematic approach to designing ClickHouse schemas for analytical workloads.

## Choose the Right Data Types

The most impactful schema decision after the primary key is using the smallest data type that correctly represents each value.

```sql
-- Prefer fixed-width integer types over generic integers
-- UInt8 (0-255) instead of Int64 for counts under 255
-- UInt32 instead of UInt64 for timestamps until 2106

-- Status codes (200, 404, 500): UInt16 uses 2 bytes vs 8 for Int64
status_code UInt16,

-- Event count per session: rarely exceeds 65,535 so UInt16 works
event_count UInt16,

-- Unix timestamp in seconds: UInt32 until year 2106 (4 bytes vs 8 for Int64)
-- Better: use DateTime which stores as UInt32 internally and formats nicely
event_time  DateTime,   -- 4 bytes, human-readable in queries

-- Millisecond timestamps: use DateTime64(3) instead of Int64
event_time_ms DateTime64(3),   -- stores as Int64 but with scale metadata

-- Floating point: prefer Float32 when full Float64 precision is not needed
-- For money values, use Decimal(18,4) to avoid floating-point rounding
value       Float32,           -- 4 bytes
amount      Decimal(18, 4),    -- exact decimal arithmetic
```

## Use LowCardinality for Repeated String Values

```sql
-- Without LowCardinality: each row stores a full string copy
service String,          -- 'api' stored as 3 bytes * 1 billion rows = 3 GB

-- With LowCardinality: strings are dictionary-encoded
service LowCardinality(String),  -- 'api' stored as a 2-byte dictionary key
                                  -- dictionary is 1 entry, ~10 bytes
                                  -- 1 billion rows: 2 GB instead of 3 GB
                                  -- Also: GROUP BY is 2-5x faster
```

Use `LowCardinality` for any string column with fewer than approximately 10,000 distinct values: status codes as strings, environment names, region codes, log levels, event types.

```sql
CREATE TABLE events
(
    event_id       UUID DEFAULT generateUUIDv4(),
    service        LowCardinality(String),
    environment    LowCardinality(String),
    log_level      LowCardinality(Enum8('debug'=1, 'info'=2, 'warn'=3, 'error'=4)),
    region         LowCardinality(FixedString(2)),
    status_code    UInt16,
    message        String,
    event_time     DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_time, service, log_level);
```

## Design the Primary Key (ORDER BY) for Your Queries

The `ORDER BY` key is the single most important schema decision. It determines:
1. How data is sorted on disk.
2. Which columns the sparse index covers for granule pruning.
3. The default sort order for range queries.

### Principle: Put the Most Selective Filter First

```sql
-- Query pattern 1: "show me all events for the last 7 days"
-- Best ORDER BY: time first (most queries filter on time)
ORDER BY (event_time, service)

-- Query pattern 2: "show me all events for service X in the last 7 days"
-- Good ORDER BY: time first if most queries span all services
ORDER BY (event_time, service)
-- OR service first if most queries filter on a specific service
ORDER BY (service, event_time)

-- Multi-tenant system: "show me events for tenant T in the last 7 days"
-- Tenant first so each tenant's data is contiguous and well-pruned
ORDER BY (tenant_id, event_time, service)
```

### Avoid Putting High-Cardinality Non-Filtered Columns in ORDER BY

```sql
-- Bad: user_id is high cardinality and rarely filtered directly
-- This causes poor data locality for time-range queries
ORDER BY (user_id, event_time)

-- Better: time first, then lower-cardinality columns
ORDER BY (event_time, service, log_level)

-- user_id can still be indexed with a skipping index if needed
ALTER TABLE events ADD INDEX idx_user user_id TYPE bloom_filter(0.01) GRANULARITY 1;
```

## Partition By Month (Default) or Day

```sql
-- Monthly partitioning (recommended for most workloads)
PARTITION BY toYYYYMM(event_time)

-- Daily partitioning (for tables where you regularly drop entire days)
PARTITION BY toDate(event_time)

-- Avoid: too fine grained (creates excessive parts under high write volume)
-- PARTITION BY toStartOfHour(event_time)  -- avoid

-- Composite partition for multi-tenant workloads
PARTITION BY (tenant_id, toYYYYMM(event_time))
```

## Use Compression Codecs Strategically

ClickHouse compresses columns by default using LZ4. For specific column patterns, specialized codecs reduce storage significantly and improve read performance (smaller data = faster reads).

```sql
CREATE TABLE metrics
(
    -- Delta + LZ4 is optimal for monotonically increasing integers (timestamps, IDs)
    event_time  DateTime CODEC(Delta, LZ4),
    metric_id   UInt64   CODEC(Delta, LZ4),

    -- DoubleDelta for counters that increase at roughly constant rates
    request_count UInt64 CODEC(DoubleDelta, LZ4),

    -- Gorilla for floating point values with slow variation (sensor readings, rates)
    cpu_usage   Float64 CODEC(Gorilla, LZ4),

    -- ZSTD(3) for high-compression needs on rarely-accessed cold columns
    -- Use ZSTD only for columns queried infrequently (it is slower to decompress)
    raw_payload String  CODEC(ZSTD(3)),

    -- LowCardinality strings compress extremely well with default LZ4
    service     LowCardinality(String)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_time, metric_id);
```

## Add Skipping Indexes for Non-Primary-Key Filters

Skipping indexes allow granule pruning on columns not in the primary key.

```sql
-- Bloom filter for equality lookups on high-cardinality string columns
ALTER TABLE events ADD INDEX idx_trace_id trace_id TYPE bloom_filter(0.01) GRANULARITY 1;

-- Minmax for range filters on numeric columns not in the primary key
ALTER TABLE events ADD INDEX idx_duration_ms duration_ms TYPE minmax GRANULARITY 4;

-- Set for low-cardinality columns with small value sets
ALTER TABLE events ADD INDEX idx_log_level log_level TYPE set(4) GRANULARITY 1;

-- Token bloom filter for full-text-style substring searches
ALTER TABLE events ADD INDEX idx_message_tokens message TYPE tokenbf_v1(32768, 3, 0) GRANULARITY 1;

-- Materialize indexes on existing data
ALTER TABLE events MATERIALIZE INDEX idx_trace_id;
ALTER TABLE events MATERIALIZE INDEX idx_duration_ms;
```

## Pre-Aggregate with Materialized Views

For queries that always aggregate the same dimensions, store pre-aggregated summaries.

```sql
-- Pre-aggregate events by hour and service
CREATE TABLE events_hourly
(
    hour         DateTime,
    service      LowCardinality(String),
    log_level    LowCardinality(String),
    event_count  UInt64,
    error_count  UInt64
)
ENGINE = SummingMergeTree(event_count, error_count)
PARTITION BY toYYYYMM(hour)
ORDER BY (hour, service, log_level);

CREATE MATERIALIZED VIEW events_hourly_mv
TO events_hourly
AS
SELECT
    toStartOfHour(event_time) AS hour,
    service,
    log_level,
    count()                   AS event_count,
    countIf(log_level = 'error') AS error_count
FROM events
GROUP BY hour, service, log_level;

-- Dashboard query reads from the tiny pre-aggregated table, not the raw events
SELECT
    hour,
    service,
    sum(event_count)  AS total_events,
    sum(error_count)  AS total_errors
FROM events_hourly
WHERE hour >= toDateTime(today() - 7)
GROUP BY hour, service
ORDER BY hour;
```

## Avoid Nullable Unless Required

`Nullable` columns add overhead: a separate bitmask column, no primary key optimization, and more complex vectorized operations.

```sql
-- Avoid when possible:
user_id Nullable(UInt64)

-- Prefer: use a sentinel value
user_id UInt64 DEFAULT 0

-- If NULL is semantically required, Nullable is acceptable
-- but be aware of the performance trade-off
```

## Complete Example Schema

```sql
CREATE TABLE application_events
(
    -- Identity
    event_id     UUID DEFAULT generateUUIDv4(),
    trace_id     String,

    -- Service dimensions (LowCardinality for fast GROUP BY)
    service      LowCardinality(String),
    environment  LowCardinality(String),
    region       LowCardinality(FixedString(2)),
    log_level    LowCardinality(Enum8('debug'=1,'info'=2,'warn'=3,'error'=4,'fatal'=5)),

    -- Metrics (smallest type that fits)
    status_code  UInt16,
    duration_ms  UInt32,
    bytes_sent   UInt32,

    -- Text fields (use ZSTD for storage efficiency on large text)
    message      String CODEC(ZSTD(1)),

    -- Time (Delta+LZ4 is optimal for monotonically increasing DateTime)
    event_time   DateTime CODEC(Delta, LZ4),

    -- Skipping indexes
    INDEX idx_trace   trace_id   TYPE bloom_filter(0.01) GRANULARITY 1,
    INDEX idx_dur     duration_ms TYPE minmax GRANULARITY 4,
    INDEX idx_msg     message    TYPE tokenbf_v1(32768, 3, 0) GRANULARITY 1
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_time, service, log_level)
TTL event_time + INTERVAL 90 DAY DELETE
SETTINGS
    index_granularity = 8192,
    min_bytes_for_wide_part = 10485760;
```

## Summary

A ClickHouse schema optimized for analytical queries requires: the right `ORDER BY` with the most-filtered column first, monthly `PARTITION BY` for time-series data, `LowCardinality` for repeated strings, the smallest correct integer types, Delta/Gorilla/ZSTD codecs matched to each column's data pattern, skipping indexes for non-primary-key filters, and materialized views for frequently aggregated dimensions. These decisions made at table creation time deliver orders-of-magnitude better query performance than adding them retroactively.
