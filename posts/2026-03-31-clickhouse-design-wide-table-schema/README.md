# How to Design a Wide Table Schema in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Wide Table, Schema Design, Denormalization, Analytics, Performance

Description: Learn how to design wide table schemas in ClickHouse for maximum analytical query performance through strategic denormalization.

---

A wide table is a fully denormalized table that combines fact and dimension data into a single row. ClickHouse is particularly well-suited for wide tables because its columnar storage reads only the columns needed by a query, making column count largely irrelevant to performance.

## What Is a Wide Table?

Instead of joining multiple normalized tables:

```text
fact_orders JOIN dim_customer JOIN dim_product JOIN dim_date
```

A wide table stores everything in one row:

```sql
CREATE TABLE orders_wide (
    -- Order facts
    order_id String,
    order_date Date,
    order_ts DateTime,
    quantity UInt32,
    revenue Decimal64(2),

    -- Customer attributes (denormalized from dim_customer)
    customer_id UInt64,
    customer_name String,
    customer_country String,
    customer_segment String,
    customer_tier String,

    -- Product attributes (denormalized from dim_product)
    product_id UInt32,
    product_name String,
    product_category String,
    product_brand String,

    -- Date attributes (denormalized from dim_date)
    order_year UInt16,
    order_quarter UInt8,
    order_month UInt8,
    order_week UInt8,
    is_weekend Bool,

    -- Pre-computed metrics
    is_high_value Bool MATERIALIZED revenue > 1000,
    revenue_bucket String MATERIALIZED
        multiIf(revenue < 10, 'micro', revenue < 100, 'small', revenue < 1000, 'medium', 'large')
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(order_date)
ORDER BY (order_date, customer_country, product_category);
```

## Advantages of Wide Tables in ClickHouse

```text
1. No JOIN overhead - all data is in one scan
2. Columnar storage ignores unused columns
3. Better compression per-column (homogeneous data)
4. Simpler query syntax
5. Easier partition pruning
```

## Practical Query Examples

Wide table queries are much simpler:

```sql
-- Revenue by country and category - no joins needed
SELECT
    customer_country,
    product_category,
    sum(revenue) AS total_revenue,
    count() AS orders
FROM orders_wide
WHERE order_year = 2025
  AND customer_segment = 'Enterprise'
GROUP BY customer_country, product_category
ORDER BY total_revenue DESC;
```

## Handling Slowly Changing Dimensions

Wide tables embed dimension data at insert time, so they capture the historical state:

```sql
-- Customer tier at the time of order (not current tier)
INSERT INTO orders_wide
SELECT
    o.order_id,
    o.order_date,
    o.customer_id,
    c.name AS customer_name,
    c.tier AS customer_tier,  -- captured at insert time
    -- ...
FROM raw_orders o
JOIN dim_customer c ON o.customer_id = c.customer_id;
```

## Managing Column Count

For tables with hundreds of columns, organize them into sections:

```sql
CREATE TABLE events_wide (
    -- Identity
    event_id String,
    timestamp DateTime,

    -- Session context
    session_id String,
    session_start DateTime,
    page_count UInt16,

    -- User context (50+ columns)
    user_id UInt64,
    user_country LowCardinality(String),
    user_plan LowCardinality(String),

    -- Device context
    device_type LowCardinality(String),
    os LowCardinality(String),
    browser LowCardinality(String),

    -- Event data
    event_type LowCardinality(String),
    properties Map(String, String)
) ENGINE = MergeTree()
ORDER BY (user_country, event_type, timestamp);
```

## When to Choose Wide Tables

```text
Use wide tables when:
- Analytical queries frequently join the same dimensions
- Query latency is the top priority
- Dimension data is stable or historical accuracy matters
- Storage costs are acceptable (extra redundancy)

Prefer star schemas when:
- Dimensions update frequently and must reflect current state
- Storage cost is a major concern
- Multiple fact tables share the same dimensions
```

## Summary

Wide tables are the natural fit for ClickHouse's columnar storage model. By denormalizing all dimension data into a single table, you eliminate join overhead and enable faster, simpler analytical queries. Use `LowCardinality` for repeated string fields to reduce storage and improve filtering performance.
