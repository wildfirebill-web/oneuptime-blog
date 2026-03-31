# Common ClickHouse Schema Design Mistakes and How to Fix Them

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Schema Design, Mistakes, Best Practice, Optimization

Description: Learn the most common ClickHouse schema design mistakes - wrong ORDER BY, missing LowCardinality, over-partitioning - and how to fix each one.

---

ClickHouse schema design mistakes are expensive to fix after data is loaded. Understanding the most common pitfalls upfront saves hours of data migration and dramatically improves query performance from day one.

## Mistake 1: Putting the Timestamp First in ORDER BY

Many developers put the timestamp first because queries filter by time range. This is backwards for ClickHouse.

```sql
-- BAD: timestamp first means ClickHouse cannot prune by user_id or country
CREATE TABLE events (
    event_time DateTime,
    country String,
    user_id UInt32
) ENGINE = MergeTree()
ORDER BY (event_time, country, user_id);

-- GOOD: low-cardinality filter columns first, timestamp last
CREATE TABLE events (
    event_time DateTime,
    country LowCardinality(String),
    user_id UInt32
) ENGINE = MergeTree()
ORDER BY (country, user_id, event_time);
```

## Mistake 2: Not Using LowCardinality

Using plain `String` for columns with few distinct values wastes memory and slows aggregations.

```sql
-- BAD: status has only 5 possible values but stored as full String
CREATE TABLE orders (
    order_id UInt64,
    status String,         -- "pending", "shipped", "delivered", "cancelled", "returned"
    country String         -- ~200 possible values
) ENGINE = MergeTree()
ORDER BY order_id;

-- GOOD: LowCardinality uses dictionary encoding
CREATE TABLE orders (
    order_id UInt64,
    status LowCardinality(String),
    country LowCardinality(String)
) ENGINE = MergeTree()
ORDER BY order_id;
```

## Mistake 3: Over-Partitioning

Partitioning by day or hour creates too many small parts, degrading performance.

```sql
-- BAD: hourly partitions create thousands of parts per month
PARTITION BY toYYYYMMDDHH(event_time)

-- GOOD: monthly partitions for most workloads
PARTITION BY toYYYYMM(event_time)

-- Only use daily if you insert 100GB+ per day
PARTITION BY toDate(event_time)
```

## Mistake 4: Using Float64 for Financial Data

Floating point cannot represent decimal values exactly, causing rounding errors in financial calculations.

```sql
-- BAD: floating point rounding errors in financial data
amount Float64

-- GOOD: exact decimal arithmetic
amount Decimal(18, 4)
```

## Mistake 5: Storing UUIDs as String

UUIDs stored as strings use 36 bytes each. As `UUID` type they use 16 bytes.

```sql
-- BAD: 36 bytes per UUID
user_id String  -- "550e8400-e29b-41d4-a716-446655440000"

-- GOOD: 16 bytes per UUID
user_id UUID
```

## Mistake 6: No Codec on Timestamp Columns

Timestamps in sorted order compress dramatically better with Delta codec.

```sql
-- BAD: no codec on sorted timestamp
event_time DateTime

-- GOOD: Delta + LZ4 compresses sorted timestamps by 10x+
event_time DateTime CODEC(Delta, LZ4)
```

## Mistake 7: Using Nullable Everywhere

`Nullable(T)` adds overhead because every value needs an extra null bit. Use only when NULL is truly a meaningful value distinct from default.

```sql
-- BAD: Nullable on a column that is always set
user_id Nullable(UInt32)  -- this is always populated

-- GOOD: Use default value when NULL is not meaningful
user_id UInt32  -- 0 means unknown/anonymous
```

## Summary

The seven most impactful ClickHouse schema mistakes are: timestamp first in ORDER BY, missing LowCardinality on string columns, over-partitioning, Float64 for money, UUIDs as strings, no Delta codec on timestamps, and overuse of Nullable. Fixing these before loading data can deliver 5-20x improvements in storage efficiency and query speed.
