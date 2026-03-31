# How to Avoid Performance Penalties of Nullable Columns in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Nullable, Performance, Schema Design, MergeTree, Optimization

Description: Learn how Nullable columns add storage and query overhead in ClickHouse and the techniques to avoid performance penalties while still handling missing data.

---

ClickHouse implements `Nullable(T)` by storing a separate null-map file alongside the data column. This extra file must be read for every query touching that column, even when the query never examines the null-map. The result is measurably higher I/O, slower filtering, and increased memory usage at query time. Avoiding unnecessary Nullable columns is one of the most impactful schema optimizations in ClickHouse.

## How Nullable Adds Overhead

For each `Nullable(T)` column ClickHouse creates two on-disk files:

```text
column.bin       -- actual data values
column.null.bin  -- null bitmask (one bit per row)
```

Every query reading that column must open and scan both files. For tables with billions of rows, even the null-map scan adds meaningful I/O.

## Benchmark: Nullable vs Non-Nullable

```sql
-- Schema with Nullable
CREATE TABLE events_nullable (
    ts       DateTime,
    user_id  Nullable(UInt64),
    country  Nullable(String),
    revenue  Nullable(Float64)
) ENGINE = MergeTree ORDER BY ts;

-- Schema without Nullable
CREATE TABLE events_notnull (
    ts       DateTime,
    user_id  UInt64     DEFAULT 0,
    country  String     DEFAULT '',
    revenue  Float64    DEFAULT 0.0
) ENGINE = MergeTree ORDER BY ts;
```

On the same dataset:
- `events_nullable` requires reading both `.bin` and `.null.bin` for each column.
- `events_notnull` reads only `.bin`, reducing I/O by 10-30% on wide tables.

## Use Sentinel Values Instead of NULL

The most straightforward avoidance pattern is to designate a sentinel value:

```sql
-- Use 0 for missing numeric IDs
user_id UInt64 DEFAULT 0

-- Use empty string for missing text
country String DEFAULT ''

-- Use -1 for missing scores
score   Int32  DEFAULT -1
```

Then filter out sentinels at query time:

```sql
SELECT avg(revenue)
FROM events_notnull
WHERE revenue != 0.0;

SELECT count(DISTINCT user_id)
FROM events_notnull
WHERE user_id != 0;
```

## Use LowCardinality Instead of Nullable(String)

For string columns with a small number of distinct values:

```sql
-- Avoid
country Nullable(String)

-- Prefer
country LowCardinality(String) DEFAULT ''
```

`LowCardinality` uses dictionary encoding, which is far more efficient than `Nullable(String)` for both storage and filtering.

## Use DEFAULT to Populate Missing Values at Insert

Prevent NULLs from entering by setting defaults at schema level:

```sql
CREATE TABLE orders (
    order_id    UInt64,
    customer_id UInt64    DEFAULT 0,
    discount    Float64   DEFAULT 0.0,
    promo_code  String    DEFAULT '',
    shipped_at  DateTime  DEFAULT toDateTime(0)
) ENGINE = MergeTree ORDER BY order_id;
```

Inserts that omit a column automatically use the default, so no NULLs are stored.

## When Nullable is Acceptable

Nullable is appropriate for:
- Columns where NULL has semantic meaning distinct from zero or empty string (e.g., optional user attributes in a CDC pipeline).
- Columns rarely queried or filtered on.
- Temporary staging tables before transformation.

Avoid Nullable for:
- Primary key components or ORDER BY columns (not allowed anyway).
- High-cardinality numeric columns used in aggregations.
- Columns in hot query paths.

## Migrating Away from Nullable

To drop Nullable from an existing column:

```sql
-- Step 1: Replace NULLs with the sentinel value
ALTER TABLE events UPDATE user_id = 0 WHERE isNull(user_id);

-- Step 2: Modify the column type
ALTER TABLE events MODIFY COLUMN user_id UInt64 DEFAULT 0;
```

Note: ALTER UPDATE runs as a mutation and is asynchronous. Monitor progress:

```sql
SELECT mutation_id, command, is_done
FROM system.mutations
WHERE table = 'events'
ORDER BY create_time DESC;
```

## Check Which Columns Are Nullable

```sql
SELECT name, type
FROM system.columns
WHERE database = 'mydb'
  AND table = 'events'
  AND startsWith(type, 'Nullable')
ORDER BY name;
```

## Summary

Nullable columns in ClickHouse carry a hidden I/O cost from the null-map file read on every query. Prefer sentinel values (0, empty string, epoch datetime) with schema defaults, and use `LowCardinality(String)` for categorical fields. Reserve `Nullable` only for columns where the distinction between NULL and a default value is semantically meaningful and the column is infrequently scanned.
