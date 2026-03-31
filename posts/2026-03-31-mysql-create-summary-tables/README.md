# How to Create Summary Tables in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Summary Table, Aggregation, Performance

Description: Learn how to build and maintain pre-aggregated summary tables in MySQL to dramatically speed up reporting queries without scanning large fact tables.

---

## What Are Summary Tables

A summary table (also called an aggregate table or rollup table) stores pre-computed aggregations of a larger fact table. Instead of running `SUM`, `COUNT`, and `AVG` across millions of rows on every report, you query the summary table which holds results already grouped by common dimensions like date, product, and region.

## When to Use Summary Tables

- Reporting queries are slow due to large fact tables
- The same aggregations are queried repeatedly
- Real-time accuracy is not required (hourly or daily staleness is acceptable)
- Analytical queries compete with OLTP traffic on the same server

## Creating a Daily Sales Summary

```sql
-- Underlying fact table
CREATE TABLE fact_sales (
  sale_id      BIGINT        NOT NULL AUTO_INCREMENT PRIMARY KEY,
  sale_date    DATE          NOT NULL,
  product_id   BIGINT        NOT NULL,
  region       VARCHAR(50)   NOT NULL,
  quantity     INT           NOT NULL,
  revenue      DECIMAL(12,2) NOT NULL,
  cost         DECIMAL(12,2) NOT NULL,
  INDEX idx_date    (sale_date),
  INDEX idx_product (product_id)
) ENGINE=InnoDB;

-- Daily summary table
CREATE TABLE summary_daily_sales (
  summary_date  DATE          NOT NULL,
  product_id    BIGINT        NOT NULL,
  region        VARCHAR(50)   NOT NULL,
  total_quantity INT          NOT NULL DEFAULT 0,
  total_revenue DECIMAL(14,2) NOT NULL DEFAULT 0.00,
  total_cost    DECIMAL(14,2) NOT NULL DEFAULT 0.00,
  order_count   INT           NOT NULL DEFAULT 0,
  updated_at    TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP
                              ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (summary_date, product_id, region)
) ENGINE=InnoDB;
```

## Populating the Summary Table

Run this as a scheduled job (e.g., via MySQL Events or an external cron):

```sql
-- Rebuild yesterday's summary (idempotent)
INSERT INTO summary_daily_sales
  (summary_date, product_id, region, total_quantity, total_revenue, total_cost, order_count)
SELECT
  sale_date,
  product_id,
  region,
  SUM(quantity)   AS total_quantity,
  SUM(revenue)    AS total_revenue,
  SUM(cost)       AS total_cost,
  COUNT(*)        AS order_count
FROM fact_sales
WHERE sale_date = CURDATE() - INTERVAL 1 DAY
GROUP BY sale_date, product_id, region
ON DUPLICATE KEY UPDATE
  total_quantity = VALUES(total_quantity),
  total_revenue  = VALUES(total_revenue),
  total_cost     = VALUES(total_cost),
  order_count    = VALUES(order_count);
```

## Automating Refresh with MySQL Events

```sql
-- Create an event scheduler job to refresh daily at 02:00
CREATE EVENT ev_refresh_daily_sales
  ON SCHEDULE EVERY 1 DAY
  STARTS (CURDATE() + INTERVAL 1 DAY + INTERVAL 2 HOUR)
  DO
    INSERT INTO summary_daily_sales
      (summary_date, product_id, region, total_quantity, total_revenue, total_cost, order_count)
    SELECT sale_date, product_id, region,
           SUM(quantity), SUM(revenue), SUM(cost), COUNT(*)
    FROM fact_sales
    WHERE sale_date = CURDATE() - INTERVAL 1 DAY
    GROUP BY sale_date, product_id, region
    ON DUPLICATE KEY UPDATE
      total_quantity = VALUES(total_quantity),
      total_revenue  = VALUES(total_revenue),
      total_cost     = VALUES(total_cost),
      order_count    = VALUES(order_count);
```

Enable the event scheduler if not already running:

```sql
SET GLOBAL event_scheduler = ON;
```

## Querying the Summary Table

```sql
-- Monthly revenue by region from the summary table (fast)
SELECT
  DATE_FORMAT(summary_date, '%Y-%m') AS month,
  region,
  SUM(total_revenue)  AS revenue,
  SUM(order_count)    AS orders
FROM summary_daily_sales
WHERE summary_date BETWEEN '2025-01-01' AND '2025-12-31'
GROUP BY month, region
ORDER BY month, revenue DESC;
```

## Incremental Updates for Intraday Use

If you need intraday summaries, maintain a running total using `INSERT ... ON DUPLICATE KEY UPDATE` each time new sales records arrive:

```sql
INSERT INTO summary_daily_sales (summary_date, product_id, region,
  total_quantity, total_revenue, total_cost, order_count)
VALUES (CURDATE(), 42, 'EMEA', 5, 250.00, 100.00, 1)
ON DUPLICATE KEY UPDATE
  total_quantity = total_quantity + VALUES(total_quantity),
  total_revenue  = total_revenue  + VALUES(total_revenue),
  total_cost     = total_cost     + VALUES(total_cost),
  order_count    = order_count    + VALUES(order_count);
```

## Summary

Summary tables trade storage for query speed by pre-computing aggregations. Build them with composite primary keys matching your most common GROUP BY columns, populate them with idempotent insert-on-duplicate-key queries, and automate the refresh with MySQL Event Scheduler. This pattern can reduce report query times from seconds to milliseconds.
