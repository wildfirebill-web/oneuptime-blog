# How to Design ClickHouse Tables for Multi-Dimensional Analysis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Multi-Dimensional Analysis, OLAP, Schema Design, Analytics

Description: Learn how to design ClickHouse tables optimized for multi-dimensional analytical queries using compound sort keys, projections, and materialized views.

---

## Multi-Dimensional Analysis in OLAP

Multi-dimensional analysis means slicing and dicing data by multiple attributes: time, geography, product category, user segment, and more. ClickHouse's columnar storage and compound sort keys make it ideal for these workloads, but schema design matters enormously for query speed.

## Fact Table Design

A well-designed fact table puts the most common filter dimensions first in the sort key:

```sql
CREATE TABLE sales_events
(
    event_time  DateTime,
    region      LowCardinality(String),
    category    LowCardinality(String),
    product_id  UInt32,
    user_id     UInt64,
    revenue     Decimal(18, 2),
    quantity    UInt16
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (region, category, product_id, event_time);
```

Queries filtering by `region` and `category` skip most of the data via the primary index.

## Skip Indexes

Add skip indexes for high-cardinality columns used in filters:

```sql
ALTER TABLE sales_events
ADD INDEX idx_user_id user_id TYPE bloom_filter(0.01) GRANULARITY 4;
```

This accelerates point lookups on `user_id` without including it in the sort key.

## Projections for Alternate Access Patterns

If queries frequently access data sorted by `product_id` first, add a projection:

```sql
ALTER TABLE sales_events
ADD PROJECTION by_product (
    SELECT * ORDER BY (product_id, event_time)
);

ALTER TABLE sales_events MATERIALIZE PROJECTION by_product;
```

ClickHouse automatically chooses the projection when the query's `ORDER BY` or `WHERE` matches.

## Aggregating Materialized Views

Pre-aggregate common rollups to speed up dashboard queries:

```sql
CREATE MATERIALIZED VIEW sales_by_region_day_mv
ENGINE = SummingMergeTree()
ORDER BY (region, day)
AS
SELECT
    region,
    toDate(event_time) AS day,
    sum(revenue) AS total_revenue,
    sum(quantity) AS total_quantity,
    count() AS order_count
FROM sales_events
GROUP BY region, day;
```

## Rollup Queries

Query multiple levels of aggregation in a single pass with `ROLLUP`:

```sql
SELECT
    region,
    category,
    sum(revenue) AS revenue
FROM sales_events
WHERE event_time >= today() - 30
GROUP BY ROLLUP(region, category)
ORDER BY region, category;
```

## Cube Analysis

Use `CUBE` for all-combinations aggregation:

```sql
SELECT
    region,
    category,
    toMonth(event_time) AS month,
    sum(revenue)
FROM sales_events
WHERE toYear(event_time) = 2026
GROUP BY CUBE(region, category, month);
```

## Summary

Designing ClickHouse tables for multi-dimensional analysis requires careful sort key selection that matches your most common filter patterns. Projections handle alternate access patterns, materialized views pre-aggregate frequent rollups, and `ROLLUP`/`CUBE` operators provide flexible aggregation without multiple queries.
