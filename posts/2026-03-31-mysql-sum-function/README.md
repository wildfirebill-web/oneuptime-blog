# How to Use the SUM() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SUM Function, Aggregate Function

Description: Learn how to use the MySQL SUM() function to total numeric values, sum within groups, conditionally sum with CASE, and compute running totals.

---

`SUM()` is a MySQL aggregate function that returns the total of all non-NULL values in a column. It is widely used for revenue calculations, inventory totals, and any metric that needs to be accumulated across rows.

## Basic SUM

```sql
-- Total revenue from all orders
SELECT SUM(total) AS total_revenue
FROM orders;

-- Total quantity of a specific product ever ordered
SELECT SUM(quantity) AS total_sold
FROM order_items
WHERE product_id = 42;
```

`SUM()` ignores NULL values - rows where the column is NULL are excluded from the sum.

## SUM with GROUP BY

```sql
-- Revenue per customer
SELECT customer_id, SUM(total) AS total_spent
FROM orders
GROUP BY customer_id
ORDER BY total_spent DESC;

-- Units sold per product
SELECT product_id, SUM(quantity) AS units_sold
FROM order_items
GROUP BY product_id
ORDER BY units_sold DESC
LIMIT 10;
```

## SUM with WHERE

```sql
-- Revenue from completed orders in 2025
SELECT SUM(total) AS revenue_2025
FROM orders
WHERE status = 'completed'
  AND YEAR(order_date) = 2025;
```

## SUM with HAVING

```sql
-- Find customers who spent more than $1000 total
SELECT customer_id, SUM(total) AS total_spent
FROM orders
WHERE status = 'completed'
GROUP BY customer_id
HAVING total_spent > 1000
ORDER BY total_spent DESC;
```

## Conditional SUM with CASE

Sum values that match a condition within a group:

```sql
-- Revenue breakdown by status per customer
SELECT
  customer_id,
  SUM(total) AS total_revenue,
  SUM(CASE WHEN status = 'completed' THEN total ELSE 0 END) AS completed_revenue,
  SUM(CASE WHEN status = 'refunded' THEN total ELSE 0 END) AS refunded_revenue
FROM orders
GROUP BY customer_id;
```

## SUM with DISTINCT

```sql
-- Sum distinct prices (unusual, but valid)
SELECT SUM(DISTINCT price) AS sum_unique_prices
FROM products
WHERE category = 'Electronics';
```

This sums each unique price value once, skipping duplicates.

## Running Totals with SUM() OVER

In MySQL 8.0+, you can compute a running total using the window function version:

```sql
SELECT
  order_date,
  total,
  SUM(total) OVER (ORDER BY order_date
                   ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
                  ) AS running_total
FROM orders
WHERE customer_id = 1
ORDER BY order_date;
```

## SUM of Expressions

```sql
-- Calculate total line value across all order items
SELECT
  order_id,
  SUM(quantity * unit_price) AS order_subtotal
FROM order_items
GROUP BY order_id;

-- Revenue with discount applied
SELECT
  order_id,
  SUM(quantity * unit_price * (1 - discount)) AS discounted_total
FROM order_items
GROUP BY order_id;
```

## Handling NULL Values

```sql
-- SUM ignores NULLs automatically
SELECT SUM(discount_amount) FROM orders;
-- Returns total of non-NULL discount amounts, not an error

-- Treat NULL as 0 explicitly with IFNULL
SELECT SUM(IFNULL(discount_amount, 0)) AS total_discounts FROM orders;
```

## Summary

`SUM()` totals non-NULL numeric values in a column, ignoring NULLs by default. Use it with `GROUP BY` to get totals per group, `HAVING` to filter groups by their sum, and `CASE WHEN` inside `SUM()` for conditional totals. For running totals and cumulative sums, use `SUM() OVER (ORDER BY ...)` as a window function in MySQL 8.0+. You can also sum computed expressions like `quantity * price` directly inside `SUM()`.
