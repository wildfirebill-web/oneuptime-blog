# How to Find Top N per Group Using Window Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Window Function, SQL, Query, Analytics

Description: Learn how to retrieve the top N rows per group in MySQL 8 using ROW_NUMBER(), RANK(), and DENSE_RANK() window functions with a filtering CTE.

---

## The Top-N-per-Group Problem

A classic SQL challenge is retrieving the top N records within each group - for example, the top 3 products by revenue per category, or the 5 highest-paid employees per department. Before MySQL 8.0, this required correlated subqueries or complicated joins. Window functions make it straightforward.

## Sample Data

```sql
CREATE TABLE products (
  product_id   INT PRIMARY KEY,
  category     VARCHAR(50),
  product_name VARCHAR(100),
  revenue      DECIMAL(12, 2)
);

INSERT INTO products VALUES
  (1, 'Electronics', 'Laptop',   120000),
  (2, 'Electronics', 'Phone',     95000),
  (3, 'Electronics', 'Tablet',    60000),
  (4, 'Electronics', 'Monitor',   45000),
  (5, 'Clothing',    'Jacket',    30000),
  (6, 'Clothing',    'Jeans',     22000),
  (7, 'Clothing',    'T-Shirt',   15000),
  (8, 'Clothing',    'Hat',        8000),
  (9, 'Books',       'Novel A',    5000),
  (10,'Books',       'Novel B',    4200),
  (11,'Books',       'Cookbook',   3800);
```

## Using ROW_NUMBER() for Strict Top-N

`ROW_NUMBER()` assigns a unique sequential number within each partition. Use it when you want exactly N rows per group, with ties broken arbitrarily:

```sql
WITH ranked AS (
  SELECT
    product_id,
    category,
    product_name,
    revenue,
    ROW_NUMBER() OVER (
      PARTITION BY category
      ORDER BY revenue DESC
    ) AS rn
  FROM products
)
SELECT product_id, category, product_name, revenue
FROM ranked
WHERE rn <= 3
ORDER BY category, rn;
```

## Using RANK() When Ties Should Repeat Positions

If two products have identical revenue, `RANK()` gives them the same rank and skips the next position. This means you may get more than N rows when ties exist:

```sql
WITH ranked AS (
  SELECT
    product_id,
    category,
    product_name,
    revenue,
    RANK() OVER (
      PARTITION BY category
      ORDER BY revenue DESC
    ) AS rnk
  FROM products
)
SELECT product_id, category, product_name, revenue, rnk
FROM ranked
WHERE rnk <= 3
ORDER BY category, rnk;
```

## Using DENSE_RANK() for Contiguous Ranks with Ties

`DENSE_RANK()` also gives ties the same rank but does not skip positions. It is useful when you want no gaps in the ranking sequence:

```sql
WITH ranked AS (
  SELECT
    product_id,
    category,
    product_name,
    revenue,
    DENSE_RANK() OVER (
      PARTITION BY category
      ORDER BY revenue DESC
    ) AS dr
  FROM products
)
SELECT product_id, category, product_name, revenue, dr
FROM ranked
WHERE dr <= 3
ORDER BY category, dr;
```

## When to Choose Each Function

| Function | Tie behavior | Rows returned for top 3 (with 2-way tie at position 3) |
|---|---|---|
| ROW_NUMBER | Arbitrary tiebreak | Exactly 3 |
| RANK | Same rank, gap after tie | 4 (positions 1,2,3,3) |
| DENSE_RANK | Same rank, no gap | 4 (positions 1,2,3,3) |

## Top-N with Multiple Sort Keys

Add secondary sort columns to the `ORDER BY` inside the window to break ties deterministically:

```sql
WITH ranked AS (
  SELECT
    *,
    ROW_NUMBER() OVER (
      PARTITION BY category
      ORDER BY revenue DESC, product_name ASC
    ) AS rn
  FROM products
)
SELECT * FROM ranked WHERE rn <= 2;
```

## Combining Top-N with Aggregates

You can combine top-N ranking with aggregated metrics:

```sql
WITH monthly_sales AS (
  SELECT
    region,
    salesperson_id,
    SUM(amount) AS total_sales
  FROM sales
  GROUP BY region, salesperson_id
),
ranked AS (
  SELECT
    *,
    RANK() OVER (PARTITION BY region ORDER BY total_sales DESC) AS rnk
  FROM monthly_sales
)
SELECT * FROM ranked WHERE rnk <= 5;
```

## Summary

Finding the top N rows per group in MySQL 8 is cleanly handled with `ROW_NUMBER()`, `RANK()`, or `DENSE_RANK()` inside a CTE, followed by a `WHERE` filter on the rank column. Choose `ROW_NUMBER()` for strict limits, `RANK()` or `DENSE_RANK()` when ties should be included at the boundary.
