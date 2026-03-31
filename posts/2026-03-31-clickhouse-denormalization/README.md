# How to Use Denormalization Effectively in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Denormalization, Schema Design, JOIN, Performance, MergeTree

Description: Learn when and how to denormalize data in ClickHouse to avoid expensive JOINs and improve analytical query performance.

---

ClickHouse is optimized for large-scale analytical queries on denormalized data. Unlike OLTP databases where normalization prevents anomalies, in ClickHouse, denormalization typically improves query performance significantly by eliminating JOINs.

## Why Denormalize in ClickHouse

ClickHouse JOINs require loading the right-side table into memory as a hash table. For large right-side tables, this is expensive and limits scalability:

```sql
-- SLOW: large right-side join
SELECT
  o.order_id,
  c.name,
  c.country,
  o.amount
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.country = 'US';
```

Denormalized version:

```sql
-- FAST: no join needed
SELECT order_id, customer_name, customer_country, amount
FROM orders_denormalized
WHERE customer_country = 'US';
```

## Flatten Dimension Tables into Fact Tables

Include dimension attributes directly in the fact table:

```sql
CREATE TABLE orders_denormalized (
  order_id UInt64,
  order_time DateTime,
  -- Denormalized from customers table
  customer_id UInt64,
  customer_name String,
  customer_country LowCardinality(String),
  customer_tier LowCardinality(String),
  -- Denormalized from products table
  product_id UInt64,
  product_name String,
  product_category LowCardinality(String),
  -- Order fields
  amount Decimal64(2),
  quantity UInt32
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(order_time)
ORDER BY (customer_country, order_time);
```

## Use Arrays for One-to-Many Relationships

Instead of a separate join table, store one-to-many relationships as arrays:

```sql
CREATE TABLE user_sessions (
  session_id UUID,
  user_id UInt64,
  start_time DateTime,
  -- Denormalized events as nested arrays
  event_types Array(LowCardinality(String)),
  event_times Array(DateTime),
  event_amounts Array(Decimal64(2))
) ENGINE = MergeTree()
ORDER BY (user_id, start_time);

-- Query without JOIN
SELECT
  user_id,
  length(event_types) AS event_count,
  arraySum(event_amounts) AS total_amount
FROM user_sessions
WHERE has(event_types, 'purchase');
```

## Use Nested Data Structures

ClickHouse's `Nested` type is syntactic sugar for paired arrays:

```sql
CREATE TABLE orders_with_items (
  order_id UInt64,
  order_time DateTime,
  items Nested(
    product_id UInt32,
    product_name String,
    quantity UInt8,
    price Decimal64(2)
  )
) ENGINE = MergeTree()
ORDER BY (order_time, order_id);
```

## Keep Small Dimension Tables Normalized

Not everything should be denormalized. Small, infrequently changing dimensions work fine as JOINs or dictionaries:

```sql
-- Use dictionary for small lookup tables
CREATE DICTIONARY country_info (
  code String,
  name String,
  region String
) PRIMARY KEY code
SOURCE(CLICKHOUSE(query 'SELECT code, name, region FROM countries'))
LAYOUT(HASHED())
LIFETIME(MIN 3600 MAX 7200);

-- Join with dictionary (fast, in-memory)
SELECT
  o.order_id,
  dictGet('country_info', 'name', customer_country) AS country_name
FROM orders o;
```

## Handle Updates to Denormalized Data

The main drawback of denormalization is that updating a dimension requires updating all fact rows. Use mutations sparingly:

```sql
-- Update customer name across all orders (expensive)
ALTER TABLE orders_denormalized
  UPDATE customer_name = 'Acme Corp Inc.'
  WHERE customer_id = 42;
```

For frequently changing attributes, consider keeping them in a dictionary instead.

## Summary

Effective denormalization in ClickHouse embeds dimension attributes directly in fact tables to eliminate JOINs, uses arrays or Nested types for one-to-many relationships, and uses dictionaries for small frequently-queried lookup tables. Reserve normalization for dimension data that changes frequently and use mutations sparingly since they are expensive in ClickHouse.
