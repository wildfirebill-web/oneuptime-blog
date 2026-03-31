# How to Use Denormalization Effectively in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Denormalization, Schema Design, Performance, JOIN, Nested

Description: Learn when and how to denormalize data in ClickHouse to eliminate expensive JOINs and maximize analytical query performance for OLAP workloads.

---

ClickHouse is optimized for denormalized schemas. Unlike OLTP databases where normalization reduces redundancy, ClickHouse queries benefit enormously from having all relevant columns in a single wide table, eliminating JOIN overhead.

## Why Denormalization Matters in ClickHouse

ClickHouse JOINs have limitations:
- The right table must fit in memory for hash joins
- ClickHouse lacks query planner join reordering (unlike PostgreSQL)
- Each JOIN adds latency for remote shard data

Denormalization eliminates these constraints by storing all data together.

## Denormalization Pattern 1: Flatten Dimensions

Instead of joining an events table with a users table, embed user attributes directly:

```sql
-- Normalized (requires JOIN at query time)
CREATE TABLE events (event_id UInt64, user_id UInt64, event_type String, event_time DateTime);
CREATE TABLE users (user_id UInt64, country String, plan_type String, signup_date Date);

-- Denormalized (no JOIN needed)
CREATE TABLE events_denorm (
    event_id UInt64,
    user_id UInt64,
    event_type LowCardinality(String),
    event_time DateTime,
    -- User attributes embedded
    user_country LowCardinality(String),
    user_plan_type LowCardinality(String),
    user_signup_date Date
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_type, event_time, user_id);
```

Query without JOIN:

```sql
-- No JOIN - directly filter and aggregate
SELECT
    user_country,
    user_plan_type,
    count() AS events
FROM events_denorm
WHERE event_time >= today() - 30
GROUP BY user_country, user_plan_type;
```

## Denormalization Pattern 2: Pre-Join at ETL Time

Use the ETL pipeline to materialize JOINs before loading:

```sql
-- In ClickHouse: denormalized insert
INSERT INTO events_denorm
SELECT
    e.event_id,
    e.user_id,
    e.event_type,
    e.event_time,
    u.country,
    u.plan_type,
    u.signup_date
FROM staging.events e
LEFT JOIN staging.users u ON e.user_id = u.user_id;
```

## Denormalization Pattern 3: Using Nested Arrays

For one-to-many relationships, use Nested columns instead of separate tables:

```sql
CREATE TABLE orders (
    order_id UInt64,
    user_id UInt64,
    order_date Date,
    -- Nested: array of items in the order
    items Nested (
        product_id UInt32,
        quantity UInt16,
        unit_price Decimal(10, 2)
    )
) ENGINE = MergeTree()
ORDER BY (user_id, order_date);
```

Query nested data:

```sql
SELECT
    order_id,
    arraySum(items.quantity) AS total_items,
    arraySum(arrayMap((q, p) -> q * p, items.quantity, items.unit_price)) AS order_total
FROM orders
WHERE order_date = today();
```

## When to Still Use JOINs

Denormalization is not always appropriate:

- **Frequently changing dimension data**: If user attributes change often, you'd need to update billions of rows. Use a dictionary instead.
- **Very wide dimensions**: If each user has 100+ attributes, embedding all in the events table wastes storage.
- **One-time analytical queries**: Occasional ad-hoc joins are fine; only optimize for recurring patterns.

## Using Dictionaries for Slowly Changing Dimensions

For slowly changing dimension data (country names, product categories), use dictionaries:

```sql
CREATE DICTIONARY user_attributes (
    user_id UInt64,
    country String,
    plan_type String
)
PRIMARY KEY user_id
SOURCE(CLICKHOUSE(TABLE 'users'))
LIFETIME(MIN 300 MAX 3600)
LAYOUT(HASHED());

-- Lookup at query time with O(1) cost
SELECT
    dictGet('user_attributes', 'country', user_id) AS country,
    count() AS events
FROM events
GROUP BY country;
```

## Measuring JOIN vs. Denormalized Performance

```sql
-- Measure JOIN approach
SELECT event_type, u.country, count()
FROM events e
JOIN users u ON e.user_id = u.user_id
GROUP BY event_type, u.country;

-- Measure denormalized approach
SELECT event_type, user_country, count()
FROM events_denorm
GROUP BY event_type, user_country;

-- Compare in query_log
SELECT query, query_duration_ms, read_rows
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_date = today()
ORDER BY event_time DESC LIMIT 5;
```

## Summary

Effective denormalization in ClickHouse embeds dimension attributes into fact tables at ETL time, uses Nested arrays for one-to-many relationships, and leverages dictionaries for slowly changing reference data. Apply denormalization selectively to your most frequent query patterns while keeping normalized staging tables for the ETL layer.
