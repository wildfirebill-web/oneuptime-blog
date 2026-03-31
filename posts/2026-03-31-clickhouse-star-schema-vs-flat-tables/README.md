# How to Use Star Schema vs Flat Tables in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Star Schema, Flat Table, Schema Design, Data Modeling, Analytics

Description: Compare star schema and flat table approaches in ClickHouse to decide which design best fits your analytical workload.

---

ClickHouse is optimized for columnar storage and analytical queries, but the schema design you choose - star schema or flat (wide) tables - has a significant impact on query performance, storage efficiency, and maintenance complexity.

## Star Schema in ClickHouse

A star schema separates facts from dimensions. You have a central fact table and multiple dimension tables joined at query time.

```sql
CREATE TABLE orders_fact (
    order_id UInt64,
    customer_id UInt32,
    product_id UInt32,
    order_date Date,
    quantity UInt16,
    revenue Decimal(18, 2)
) ENGINE = MergeTree()
ORDER BY (order_date, customer_id);

CREATE TABLE customers_dim (
    customer_id UInt32,
    customer_name String,
    country String,
    segment String
) ENGINE = MergeTree()
ORDER BY customer_id;
```

Querying with a join:

```sql
SELECT
    c.country,
    sum(f.revenue) AS total_revenue
FROM orders_fact f
JOIN customers_dim c ON f.customer_id = c.customer_id
WHERE f.order_date >= '2025-01-01'
GROUP BY c.country
ORDER BY total_revenue DESC;
```

Star schemas work well in ClickHouse when dimensions are large and reused across many fact tables.

## Flat Tables in ClickHouse

Flat tables denormalize everything into a single wide table, eliminating joins entirely.

```sql
CREATE TABLE orders_flat (
    order_id UInt64,
    order_date Date,
    quantity UInt16,
    revenue Decimal(18, 2),
    customer_id UInt32,
    customer_name String,
    country LowCardinality(String),
    segment LowCardinality(String),
    product_id UInt32,
    product_name String,
    category LowCardinality(String)
) ENGINE = MergeTree()
ORDER BY (order_date, country, category);
```

Querying is simpler and faster because no joins are required:

```sql
SELECT
    country,
    sum(revenue) AS total_revenue
FROM orders_flat
WHERE order_date >= '2025-01-01'
GROUP BY country
ORDER BY total_revenue DESC;
```

## Performance Comparison

ClickHouse performs joins less efficiently than traditional RDBMS due to its distributed and columnar nature. Flat tables tend to outperform star schemas for most read-heavy analytical workloads because:

- No join overhead at query time
- Better compression on repeated low-cardinality strings with `LowCardinality`
- Simpler query planning and execution

## When to Use Each Approach

Use star schemas when:
- Dimension tables are large and frequently updated independently
- Multiple fact tables share the same dimensions
- Storage space is a concern and deduplication matters

Use flat tables when:
- Query latency is the top priority
- The dataset is append-only or rarely updated
- Dimensions are small and low-cardinality

## Hybrid Approach

A practical middle ground is to use flat tables for the hot query path and maintain separate dimension tables for updates:

```sql
-- Refresh flat table with latest dimension data
INSERT INTO orders_flat
SELECT
    f.order_id, f.order_date, f.quantity, f.revenue,
    f.customer_id, c.customer_name, c.country, c.segment,
    f.product_id, p.product_name, p.category
FROM orders_fact f
JOIN customers_dim c ON f.customer_id = c.customer_id
JOIN products_dim p ON f.product_id = p.product_id;
```

## Summary

In ClickHouse, flat tables generally offer better query performance due to avoided join costs. Star schemas remain useful when dimensions are large, mutable, or shared. For most analytics workloads, starting with a flat denormalized table and using `LowCardinality` for repeated string fields is the recommended approach.
