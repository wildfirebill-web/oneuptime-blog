# How to Use MySQL for Real-Time Analytics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Analytics, Performance, Index, Aggregation

Description: Learn how to run real-time analytics queries on MySQL using covering indexes, summary tables, and materialized aggregations to support dashboards without degrading OLTP performance.

---

MySQL is an OLTP database, but many applications need real-time analytics alongside transactional workloads. With the right schema design and query patterns, MySQL can serve dashboards and aggregation queries without requiring a separate analytics database.

## Separate Reads from Writes

The first step is routing analytics queries to a read replica so they do not compete with OLTP writes on the primary:

```python
write_engine = create_engine("mysql+pymysql://user:pass@primary/myapp")
analytics_engine = create_engine("mysql+pymysql://user:pass@replica/myapp")
```

Read replicas add 1-2 seconds of replication lag, which is acceptable for most dashboard use cases.

## Use Covering Indexes for Aggregation Queries

Analytics queries typically filter on a time range and group by a dimension. A covering index that includes the filter and group-by columns enables the aggregation to run entirely from the index:

```sql
-- Index covers the WHERE and GROUP BY columns
ALTER TABLE orders ADD INDEX idx_analytics (created_at, status, total_amount);

-- This query uses the covering index - no row reads
SELECT
  DATE(created_at) AS order_date,
  status,
  COUNT(*) AS order_count,
  SUM(total_amount) AS revenue
FROM orders
WHERE created_at >= '2026-01-01' AND created_at < '2026-04-01'
GROUP BY order_date, status
ORDER BY order_date;
```

## Pre-Aggregate with Summary Tables

For metrics queried thousands of times per day, compute aggregations once and store results:

```sql
CREATE TABLE daily_order_summary (
  summary_date DATE NOT NULL,
  status       VARCHAR(50) NOT NULL,
  order_count  INT UNSIGNED NOT NULL,
  revenue      DECIMAL(14,2) NOT NULL,
  PRIMARY KEY (summary_date, status)
) ENGINE=InnoDB;
```

Populate the summary table with a scheduled job:

```sql
INSERT INTO daily_order_summary (summary_date, status, order_count, revenue)
SELECT
  DATE(created_at),
  status,
  COUNT(*),
  SUM(total_amount)
FROM orders
WHERE DATE(created_at) = CURDATE() - INTERVAL 1 DAY
GROUP BY DATE(created_at), status
ON DUPLICATE KEY UPDATE
  order_count = VALUES(order_count),
  revenue     = VALUES(revenue);
```

Dashboard queries hit the summary table instead of the raw orders table:

```sql
SELECT summary_date, SUM(revenue) AS daily_revenue
FROM daily_order_summary
WHERE summary_date >= '2026-01-01'
GROUP BY summary_date
ORDER BY summary_date;
```

## Use Window Functions for Running Totals

MySQL 8.0 window functions enable cumulative analytics directly in SQL:

```sql
SELECT
  summary_date,
  SUM(revenue) AS daily_revenue,
  SUM(SUM(revenue)) OVER (ORDER BY summary_date) AS cumulative_revenue
FROM daily_order_summary
WHERE summary_date >= '2026-03-01'
GROUP BY summary_date
ORDER BY summary_date;
```

## Dashboard Query Pattern with JSON Output

Return dashboard data in a single query using JSON aggregation for the frontend:

```sql
SELECT
  JSON_OBJECT(
    'date', summary_date,
    'revenue', SUM(revenue),
    'order_count', SUM(order_count)
  ) AS day_stats
FROM daily_order_summary
WHERE summary_date >= CURDATE() - INTERVAL 30 DAY
GROUP BY summary_date
ORDER BY summary_date;
```

## Summary

MySQL real-time analytics becomes practical when you route queries to a read replica, use covering indexes for raw aggregations, pre-compute summary tables for frequently-accessed metrics, and leverage window functions for running totals. This layered approach keeps OLTP performance unaffected while delivering sub-second dashboard response times. For sub-second analytics on billions of rows, a columnar database like ClickHouse or BigQuery is a better fit.
