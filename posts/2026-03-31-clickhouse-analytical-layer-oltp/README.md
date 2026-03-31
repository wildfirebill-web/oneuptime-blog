# How to Use ClickHouse as an Analytical Layer Over OLTP Databases

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, OLAP, OLTP, Analytics, Architecture, PostgreSQL, MySQL

Description: Use ClickHouse as a dedicated analytical layer on top of your OLTP databases to offload complex queries and enable real-time business intelligence.

---

## Overview

OLTP databases like PostgreSQL and MySQL are optimized for transactional workloads - short reads and writes against individual rows. Running complex analytical queries (aggregations, group-bys, time-series) on OLTP systems creates resource contention and slows down your application.

Adding ClickHouse as a dedicated analytical layer separates concerns: OLTP databases handle transactional writes, ClickHouse handles analytical reads.

## Architecture

```text
Application -> PostgreSQL / MySQL (OLTP: writes + transactional reads)
                     |
                     | (CDC via Debezium or MaterializedDB)
                     v
              ClickHouse (OLAP: analytical queries, dashboards, reporting)
                     |
                     v
              BI Tools (Grafana, Metabase, Superset, Tableau)
```

## Step 1: Connect ClickHouse to MySQL

Use the `MaterializedMySQL` engine for a live replica:

```sql
CREATE DATABASE mysql_analytics
ENGINE = MaterializedMySQL('mysql-host:3306', 'app_db', 'ch_reader', 'pass')
SETTINGS materialized_mysql_tables_list = 'orders,customers,products';
```

All tables in `app_db` are available in ClickHouse, updated in real-time from the binlog.

## Step 2: Connect ClickHouse to PostgreSQL

Use the `MaterializedPostgreSQL` engine for a live replica:

```sql
CREATE DATABASE pg_analytics
ENGINE = MaterializedPostgreSQL('pg-host:5432', 'app_db', 'ch_reader', 'pass')
SETTINGS materialized_postgresql_tables_list = 'orders,customers';
```

## Step 3: Build a Denormalized Analytics Table

Denormalize data from multiple sources into a single wide table for fast queries:

```sql
CREATE TABLE orders_wide AS
SELECT
    o.order_id,
    o.amount,
    o.status,
    o.created_at,
    c.name       AS customer_name,
    c.country    AS customer_country,
    c.tier       AS customer_tier,
    p.category   AS product_category,
    p.brand      AS product_brand
FROM mysql_analytics.orders   AS o
JOIN mysql_analytics.customers AS c ON o.customer_id = c.customer_id
JOIN mysql_analytics.products  AS p ON o.product_id = p.product_id;
```

Persist the result locally:

```sql
CREATE TABLE orders_flat (
    order_id          String,
    amount            Decimal(18, 2),
    status            LowCardinality(String),
    created_at        DateTime,
    customer_name     String,
    customer_country  LowCardinality(String),
    customer_tier     LowCardinality(String),
    product_category  LowCardinality(String),
    product_brand     LowCardinality(String)
) ENGINE = MergeTree()
ORDER BY (customer_country, created_at);
```

## Step 4: Answer Business Questions Fast

Revenue by customer tier this month:

```sql
SELECT
    customer_tier,
    count()      AS num_orders,
    sum(amount)  AS revenue
FROM orders_flat
WHERE created_at >= toStartOfMonth(today())
GROUP BY customer_tier
ORDER BY revenue DESC;
```

Month-over-month growth by product category:

```sql
SELECT
    product_category,
    sumIf(amount, toYYYYMM(created_at) = toYYYYMM(today()))       AS current_month,
    sumIf(amount, toYYYYMM(created_at) = toYYYYMM(today() - 32))  AS prev_month,
    round((current_month - prev_month) / prev_month * 100, 2)      AS growth_pct
FROM orders_flat
WHERE created_at >= toStartOfMonth(today() - 32)
GROUP BY product_category
ORDER BY current_month DESC;
```

## Step 5: Offload Report Queries

Instead of running heavy reports against PostgreSQL at month-end, schedule them in ClickHouse:

```bash
clickhouse-client --query="
    SELECT customer_country, sum(amount), count()
    FROM orders_flat
    WHERE created_at >= toStartOfMonth(today())
    GROUP BY customer_country
    FORMAT CSV
" > /reports/monthly_by_country.csv
```

## Summary

ClickHouse as an analytical layer separates reporting workloads from transactional operations. Use MaterializedMySQL or MaterializedPostgreSQL to keep data in sync, build denormalized flat tables for common query patterns, and let your BI tools query ClickHouse instead of your production OLTP database.
