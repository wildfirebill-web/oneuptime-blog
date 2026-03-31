# How to Create Tables with Multiple Indexes in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, DDL, Index, Primary Key, Secondary Index, Skipping Index

Description: Learn how to define primary keys, ORDER BY keys, and data skipping indexes in ClickHouse CREATE TABLE for faster query performance.

---

ClickHouse uses a sparse primary index based on the `ORDER BY` key, and optionally supports data skipping indexes for selective filtering on non-key columns. Understanding how these two index layers work together is key to designing high-performance ClickHouse tables.

## Primary Key vs ORDER BY

In ClickHouse, `ORDER BY` determines both the physical sort order of data on disk and, by default, the primary key. The primary key is a prefix of `ORDER BY` and defines the sparse index stored in `primary.idx`.

```sql
-- ORDER BY is the primary key by default
CREATE TABLE events
(
    created_at DateTime,
    user_id    UInt64,
    type       LowCardinality(String),
    payload    String
)
ENGINE = MergeTree()
ORDER BY (created_at, user_id);
-- Primary key = (created_at, user_id)
```

A separate `PRIMARY KEY` can be set as a shorter prefix of `ORDER BY` when you need a different deduplication key (e.g., in ReplacingMergeTree):

```sql
CREATE TABLE user_events
(
    user_id    UInt64,
    event_id   UInt64,
    created_at DateTime,
    type       LowCardinality(String)
)
ENGINE = ReplacingMergeTree()
-- Deduplication uses only user_id + event_id
PRIMARY KEY (user_id, event_id)
-- Physical sort order includes created_at for range queries
ORDER BY (user_id, event_id, created_at);
```

## Data Skipping Indexes

Data skipping indexes are secondary indexes stored alongside granules (blocks of ~8192 rows by default). When a query filter cannot use the primary key, a skipping index can skip entire granules that cannot contain matching rows.

### INDEX Clause Syntax

```sql
INDEX index_name column_expr TYPE index_type(params) GRANULARITY n
```

`GRANULARITY` sets how many primary key granules are combined into one index entry. Higher values produce a smaller index but skip fewer granules.

## minmax Index

Stores the minimum and maximum value of a column per index granule. Best for range filters on numeric or date columns that are weakly correlated with ORDER BY.

```sql
CREATE TABLE orders
(
    order_id    UInt64,
    created_at  DateTime,
    user_id     UInt64,
    amount      Float64,
    status      LowCardinality(String),
    INDEX idx_amount amount TYPE minmax GRANULARITY 4,
    INDEX idx_created created_at TYPE minmax GRANULARITY 2
)
ENGINE = MergeTree()
ORDER BY order_id;
```

## set Index

Stores the unique set of values per granule. Best for low-cardinality columns filtered by equality or IN.

```sql
CREATE TABLE page_views
(
    ts          DateTime,
    page_path   String,
    user_id     UInt64,
    country     LowCardinality(String),
    INDEX idx_country country TYPE set(100) GRANULARITY 4,
    INDEX idx_page    page_path TYPE set(0) GRANULARITY 2
)
ENGINE = MergeTree()
ORDER BY (ts, user_id);
-- set(0) means unlimited unique values stored per granule
-- set(N) caps at N unique values; if exceeded, the granule is not skipped
```

## bloom_filter Index

A probabilistic filter that allows fast equality lookups on string or numeric columns with high cardinality.

```sql
CREATE TABLE traces
(
    trace_id    String,
    span_id     String,
    started_at  DateTime64(3),
    service     LowCardinality(String),
    status_code UInt16,
    INDEX idx_trace_id  trace_id  TYPE bloom_filter(0.01) GRANULARITY 1,
    INDEX idx_span_id   span_id   TYPE bloom_filter(0.01) GRANULARITY 1
)
ENGINE = MergeTree()
ORDER BY started_at;
-- 0.01 = 1% false positive rate
```

## tokenbf_v1 Index

A token Bloom filter that splits strings into tokens (words) and indexes them individually. Best for full-text-style substring or LIKE queries.

```sql
CREATE TABLE application_logs
(
    ts       DateTime,
    service  LowCardinality(String),
    level    LowCardinality(String),
    message  String,
    INDEX idx_message_tokens message
        TYPE tokenbf_v1(32768, 3, 0)
        GRANULARITY 1
)
ENGINE = MergeTree()
ORDER BY (ts, service);
-- tokenbf_v1(size_of_bloom_filter_in_bytes, number_of_hash_functions, random_seed)
```

## ngrambf_v1 Index

Similar to `tokenbf_v1` but indexes character n-grams instead of whitespace-split tokens. Good for partial string matching without word boundaries.

```sql
CREATE TABLE product_catalog
(
    product_id  UInt64,
    name        String,
    description String,
    INDEX idx_name_ngram name
        TYPE ngrambf_v1(4, 32768, 3, 0)
        GRANULARITY 1
)
ENGINE = MergeTree()
ORDER BY product_id;
-- ngrambf_v1(ngram_size, filter_size_bytes, hash_functions, seed)
```

## Complete Table with Multiple Index Types

```sql
CREATE TABLE web_events
(
    -- Columns
    event_id     UUID            DEFAULT generateUUIDv4(),
    created_at   DateTime64(3)   DEFAULT now64(),
    user_id      UInt64,
    session_id   String,
    page_url     String,
    referrer     String,
    country      LowCardinality(String),
    device_type  LowCardinality(String),
    response_ms  UInt32,
    status_code  UInt16,

    -- Skipping indexes
    INDEX idx_user_id     user_id      TYPE minmax          GRANULARITY 8,
    INDEX idx_response    response_ms  TYPE minmax          GRANULARITY 4,
    INDEX idx_status      status_code  TYPE set(20)         GRANULARITY 4,
    INDEX idx_country     country      TYPE set(250)        GRANULARITY 4,
    INDEX idx_session     session_id   TYPE bloom_filter(0.01) GRANULARITY 1,
    INDEX idx_url_tokens  page_url     TYPE tokenbf_v1(32768, 3, 0) GRANULARITY 1
)
ENGINE = MergeTree()
PARTITION BY toDate(created_at)
ORDER BY (created_at, user_id)
SETTINGS index_granularity = 8192;
```

## Adding Indexes to an Existing Table

```sql
-- Add a skipping index
ALTER TABLE web_events
    ADD INDEX idx_device device_type TYPE set(10) GRANULARITY 4;

-- Materialise the index for existing data
ALTER TABLE web_events MATERIALIZE INDEX idx_device;

-- Drop an index
ALTER TABLE web_events DROP INDEX idx_device;
```

## Checking Index Usage with EXPLAIN

```sql
-- See which indexes the query planner uses
EXPLAIN indexes = 1
SELECT *
FROM web_events
WHERE status_code = 500
  AND created_at >= now() - INTERVAL 1 DAY;
```

## Summary

ClickHouse tables rely on the sparse primary index derived from `ORDER BY` for efficient range scans on key columns. Data skipping indexes extend filtering capability to non-key columns: `minmax` suits range queries, `set` suits equality/IN filters on low-cardinality data, `bloom_filter` handles high-cardinality equality lookups, and `tokenbf_v1`/`ngrambf_v1` accelerate text searches. Combining these index types on a single table allows ClickHouse to skip large portions of data across diverse query patterns.
