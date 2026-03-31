# How to Create Crosstab Queries in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Crosstab, Pivot, CASE, Report

Description: Learn how to create crosstab (pivot) queries in MySQL using CASE expressions and GROUP BY to transform rows into columns for cross-dimensional reporting.

---

## What Is a Crosstab Query?

A crosstab (also called a pivot table) transforms row values into columns. For example, displaying monthly revenue as separate columns (Jan, Feb, Mar...) instead of separate rows. MySQL does not have a native PIVOT keyword, but `CASE` expressions with `GROUP BY` achieve the same result.

## Static Crosstab with CASE

Transform monthly revenue rows into year columns:

```sql
SELECT
  MONTH(order_date) AS month_num,
  MONTHNAME(MIN(order_date)) AS month_name,
  SUM(CASE WHEN YEAR(order_date) = 2023 THEN total_amount ELSE 0 END) AS revenue_2023,
  SUM(CASE WHEN YEAR(order_date) = 2024 THEN total_amount ELSE 0 END) AS revenue_2024,
  SUM(CASE WHEN YEAR(order_date) = 2025 THEN total_amount ELSE 0 END) AS revenue_2025
FROM orders
GROUP BY MONTH(order_date)
ORDER BY month_num;
```

## Crosstab: Days of Week vs. Hour

Show order counts by day of week and hour of day:

```sql
SELECT
  DAYNAME(order_date) AS day_of_week,
  SUM(CASE WHEN HOUR(order_date) BETWEEN 0  AND 5  THEN 1 ELSE 0 END) AS midnight_to_6am,
  SUM(CASE WHEN HOUR(order_date) BETWEEN 6  AND 11 THEN 1 ELSE 0 END) AS morning,
  SUM(CASE WHEN HOUR(order_date) BETWEEN 12 AND 17 THEN 1 ELSE 0 END) AS afternoon,
  SUM(CASE WHEN HOUR(order_date) BETWEEN 18 AND 23 THEN 1 ELSE 0 END) AS evening
FROM orders
GROUP BY DAYOFWEEK(order_date), DAYNAME(order_date)
ORDER BY DAYOFWEEK(order_date);
```

## Category vs. Month Crosstab

Revenue per product category pivoted by month:

```sql
SELECT
  p.category,
  SUM(CASE WHEN MONTH(o.order_date) = 1  THEN oi.quantity * oi.unit_price ELSE 0 END) AS jan,
  SUM(CASE WHEN MONTH(o.order_date) = 2  THEN oi.quantity * oi.unit_price ELSE 0 END) AS feb,
  SUM(CASE WHEN MONTH(o.order_date) = 3  THEN oi.quantity * oi.unit_price ELSE 0 END) AS mar,
  SUM(CASE WHEN MONTH(o.order_date) = 4  THEN oi.quantity * oi.unit_price ELSE 0 END) AS apr,
  SUM(CASE WHEN MONTH(o.order_date) = 5  THEN oi.quantity * oi.unit_price ELSE 0 END) AS may,
  SUM(CASE WHEN MONTH(o.order_date) = 6  THEN oi.quantity * oi.unit_price ELSE 0 END) AS jun,
  SUM(CASE WHEN MONTH(o.order_date) = 7  THEN oi.quantity * oi.unit_price ELSE 0 END) AS jul,
  SUM(CASE WHEN MONTH(o.order_date) = 8  THEN oi.quantity * oi.unit_price ELSE 0 END) AS aug,
  SUM(CASE WHEN MONTH(o.order_date) = 9  THEN oi.quantity * oi.unit_price ELSE 0 END) AS sep,
  SUM(CASE WHEN MONTH(o.order_date) = 10 THEN oi.quantity * oi.unit_price ELSE 0 END) AS oct,
  SUM(CASE WHEN MONTH(o.order_date) = 11 THEN oi.quantity * oi.unit_price ELSE 0 END) AS nov,
  SUM(CASE WHEN MONTH(o.order_date) = 12 THEN oi.quantity * oi.unit_price ELSE 0 END) AS dec
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE YEAR(o.order_date) = 2025
GROUP BY p.category
ORDER BY p.category;
```

## Dynamic Crosstab with Prepared Statements

When column values are not known in advance, build the query dynamically:

```sql
SET @sql = NULL;
SELECT
  GROUP_CONCAT(DISTINCT
    CONCAT(
      'SUM(CASE WHEN YEAR(order_date) = ', yr, ' THEN total_amount ELSE 0 END) AS `', yr, '`'
    )
    ORDER BY yr
  ) INTO @sql
FROM (SELECT DISTINCT YEAR(order_date) AS yr FROM orders) years;

SET @sql = CONCAT('SELECT MONTH(order_date) AS month, ', @sql,
                  ' FROM orders GROUP BY MONTH(order_date) ORDER BY month');

PREPARE stmt FROM @sql;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
```

## Summary

Create crosstab queries in MySQL using `CASE WHEN column_value = 'X' THEN metric ELSE 0 END` combined with `SUM` and `GROUP BY`. Apply it to pivot revenue by month, orders by time of day, or category by product. For dynamic pivots with unknown column values, use prepared statements with `GROUP_CONCAT` to generate the query at runtime.
