# How to Use ClickHouse as an Analytical Layer Over OLTP Databases

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, OLTP, OLAP, Architecture, MySQL, PostgreSQL, Analytics

Description: Build an analytical layer using ClickHouse on top of OLTP databases to offload analytical queries and improve performance.

---

OLTP databases like MySQL and PostgreSQL are optimized for transactional workloads, not analytical queries. Adding ClickHouse as an analytical layer enables complex aggregations and reports without impacting transactional performance.

## Architecture Pattern

```text
Application --> MySQL/PostgreSQL (OLTP: writes + simple reads)
                     |
               CDC / Replication
                     |
                     v
            ClickHouse (OLAP: analytical queries, dashboards, reports)
                     |
                     v
            BI Tools (Grafana, Metabase, Superset)
```

## Real-Time Replication Layer

Use `MaterializedMySQL` or `MaterializedPostgreSQL` for real-time sync:

```sql
-- MySQL
CREATE DATABASE mysql_analytics
ENGINE = MaterializedMySQL('mysql-host:3306', 'myapp', 'user', 'secret');

-- PostgreSQL
CREATE DATABASE pg_analytics
ENGINE = MaterializedPostgreSQL('pg-host:5432', 'myapp', 'user', 'secret');
```

## Dedicated Analytical Tables

Create denormalized analytical tables for dashboard queries:

```sql
CREATE TABLE orders_analytics (
    order_id UInt64,
    created_at DateTime,
    status LowCardinality(String),
    total_amount Decimal(18, 4),
    customer_country LowCardinality(String),
    product_category LowCardinality(String),
    discount_applied UInt8
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (status, created_at, customer_country);
```

Populate from the replicated OLTP data:

```sql
INSERT INTO orders_analytics
SELECT
    o.id AS order_id,
    o.created_at,
    o.status,
    o.total_amount,
    c.country AS customer_country,
    p.category AS product_category,
    if(o.discount_code IS NOT NULL, 1, 0) AS discount_applied
FROM mysql_analytics.orders o
JOIN mysql_analytics.customers c ON o.customer_id = c.id
JOIN mysql_analytics.products p ON o.product_id = p.id
WHERE o.created_at >= yesterday();
```

## Pre-Aggregated Summary Tables

For dashboards with repeated queries, maintain pre-aggregated tables:

```sql
CREATE TABLE daily_revenue (
    date Date,
    country LowCardinality(String),
    category LowCardinality(String),
    total_revenue Decimal(18, 4),
    order_count UInt32,
    unique_customers UInt32
) ENGINE = SummingMergeTree((total_revenue, order_count, unique_customers))
ORDER BY (date, country, category);
```

## Offload Reporting Queries

Run heavy reports in ClickHouse instead of MySQL:

```sql
-- ClickHouse: runs in milliseconds
SELECT
    toStartOfMonth(created_at) AS month,
    customer_country,
    sum(total_amount) AS revenue,
    count() AS orders,
    uniq(order_id) AS unique_orders
FROM orders_analytics
WHERE created_at >= now() - INTERVAL 12 MONTH
GROUP BY month, customer_country
ORDER BY month DESC, revenue DESC;
```

The equivalent MySQL query might take minutes on a large table.

## Read Routing Strategy

Configure your application to route queries:

```python
def get_db_connection(query_type):
    if query_type == 'transactional':
        return mysql_connection
    elif query_type == 'analytical':
        return clickhouse_connection
```

## Summary

ClickHouse as an analytical layer over OLTP databases separates read workloads from transactional operations. Use CDC replication for real-time data availability, denormalize data for analytical performance, and pre-aggregate for dashboard queries. This pattern scales analytical workloads independently from OLTP without schema changes to production databases.
