# How to Use DISTINCT to Remove Duplicate Rows in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Distinct, Deduplication

Description: Learn how to use MySQL DISTINCT to remove duplicate rows from query results, including multi-column deduplication, use with aggregate functions, and performance tips.

---

`SELECT DISTINCT` tells MySQL to return only unique combinations of the selected columns, filtering out duplicate rows from the result set. It is useful when joins or denormalized data would otherwise produce repeated rows.

## Basic DISTINCT on One Column

```sql
-- Return each unique department name once
SELECT DISTINCT department
FROM employees
ORDER BY department;
```

Without `DISTINCT`, if 50 employees are in Engineering, you would get 50 rows with "Engineering". With `DISTINCT`, you get it once.

## DISTINCT on Multiple Columns

`DISTINCT` applies to the combination of all selected columns:

```sql
-- Unique combinations of city and country
SELECT DISTINCT city, country
FROM customers
ORDER BY country, city;
```

A row is only removed if all selected columns match another row exactly. Two rows with the same city but different countries are both included.

## DISTINCT with WHERE

```sql
-- Unique product categories that have products under $50
SELECT DISTINCT category
FROM products
WHERE price < 50.00;
```

## DISTINCT with ORDER BY

```sql
-- Unique statuses used in orders, sorted
SELECT DISTINCT status
FROM orders
ORDER BY status ASC;
```

## DISTINCT with Aggregate Functions

`DISTINCT` can be used inside aggregate functions to count or sum unique values:

```sql
-- Count distinct customers who placed orders in 2025
SELECT COUNT(DISTINCT customer_id) AS unique_customers
FROM orders
WHERE YEAR(order_date) = 2025;

-- Sum distinct product prices (unusual but valid)
SELECT SUM(DISTINCT price) AS sum_of_unique_prices
FROM products
WHERE category = 'Electronics';
```

## DISTINCT vs GROUP BY

`GROUP BY` without an aggregate function produces the same result as `DISTINCT`:

```sql
-- These are equivalent
SELECT DISTINCT department FROM employees;
SELECT department FROM employees GROUP BY department;
```

However, `GROUP BY` is more explicit when you plan to add aggregates later. `DISTINCT` is cleaner for pure deduplication.

## DISTINCT and NULL

NULL values are treated as equal by `DISTINCT` - multiple NULL values collapse into one:

```sql
-- If manager_id has three NULL rows, only one NULL is returned
SELECT DISTINCT manager_id FROM employees ORDER BY manager_id;
```

## Performance Considerations

`DISTINCT` requires MySQL to sort or hash all result rows to identify duplicates, which adds overhead:

```sql
-- Check query plan for DISTINCT
EXPLAIN SELECT DISTINCT customer_id
FROM orders
WHERE order_date >= '2025-01-01'\G
```

If the column has an index, MySQL may use it to return distinct values efficiently. For large result sets, `DISTINCT` on an unindexed column can be slow. Consider whether your query actually produces duplicates - sometimes they can be avoided by restructuring the join rather than adding `DISTINCT`.

## Summary

`SELECT DISTINCT` eliminates duplicate rows based on the full combination of selected columns. It works on one or multiple columns, and can be used inside aggregate functions like `COUNT(DISTINCT col)`. NULL values collapse to a single NULL under `DISTINCT`. It is semantically equivalent to `GROUP BY` without aggregates, but `GROUP BY` is preferred when you intend to add aggregate functions. For large datasets, verify with `EXPLAIN` that an index is being used to avoid expensive sort-based deduplication.
