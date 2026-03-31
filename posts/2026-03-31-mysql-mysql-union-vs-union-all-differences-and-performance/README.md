# MySQL UNION vs UNION ALL: Differences and Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, UNION, UNION ALL, SQL, Query Optimization, Performance

Description: Learn the difference between MySQL UNION and UNION ALL, when each is appropriate, and how they affect query performance.

---

## What Is UNION

`UNION` combines the result sets of two or more SELECT statements into a single result set, automatically removing duplicate rows. The result is equivalent to performing a UNION ALL followed by a DISTINCT operation.

```sql
SELECT city FROM customers
UNION
SELECT city FROM suppliers;
```

This returns each distinct city appearing in either table, with no duplicates.

## What Is UNION ALL

`UNION ALL` combines result sets without removing duplicates. Every row from every SELECT statement appears in the output, even if identical rows exist.

```sql
SELECT city FROM customers
UNION ALL
SELECT city FROM suppliers;
```

This returns every city from both tables, including repeats.

## Key Rules for UNION and UNION ALL

Both operators require:
1. The same number of columns in each SELECT
2. Compatible data types in corresponding columns
3. Column names from the first SELECT are used in the result

```sql
-- Error: different number of columns
SELECT id, name FROM customers
UNION
SELECT id FROM suppliers;

-- Works: compatible types
SELECT id, name FROM customers
UNION ALL
SELECT order_id, customer_name FROM orders;
```

## Performance Difference

`UNION` is slower than `UNION ALL` because it must perform a deduplication step after combining the result sets. MySQL typically does this by sorting the combined results or using a hash-based approach to detect duplicates.

`UNION ALL` simply appends the result sets together with no extra processing.

```sql
-- Faster: no deduplication needed
SELECT product_id FROM order_items
UNION ALL
SELECT product_id FROM wishlist_items;

-- Slower: must deduplicate
SELECT product_id FROM order_items
UNION
SELECT product_id FROM wishlist_items;
```

For large result sets, the performance difference can be significant. Always prefer `UNION ALL` unless you specifically need deduplication.

## When to Use UNION

Use `UNION` when duplicate rows would produce incorrect results:

```sql
-- Get unique email addresses from both customers and staff tables
SELECT email FROM customers
UNION
SELECT email FROM staff;
```

```sql
-- List all unique product categories across multiple catalogs
SELECT category FROM catalog_2024
UNION
SELECT category FROM catalog_2025;
```

## When to Use UNION ALL

Use `UNION ALL` when:
- Duplicates are acceptable or impossible
- You plan to aggregate the combined data
- You are combining non-overlapping partitions

```sql
-- Aggregate across partitioned tables - use UNION ALL for correctness and speed
SELECT SUM(revenue) FROM (
  SELECT revenue FROM sales_q1
  UNION ALL
  SELECT revenue FROM sales_q2
  UNION ALL
  SELECT revenue FROM sales_q3
  UNION ALL
  SELECT revenue FROM sales_q4
) AS all_sales;
```

Using `UNION` here would produce wrong totals if any revenue values happened to match across quarters.

## ORDER BY with UNION

To sort the final combined result, apply ORDER BY to the last SELECT:

```sql
SELECT id, name, 'customer' AS type FROM customers
UNION ALL
SELECT id, name, 'supplier' AS type FROM suppliers
ORDER BY name ASC;
```

You cannot use ORDER BY inside individual SELECT statements within a UNION unless they are wrapped in a subquery:

```sql
-- Sort within each subset before combining
SELECT * FROM (
  SELECT id, name FROM customers ORDER BY name LIMIT 10
) AS top_customers
UNION ALL
SELECT * FROM (
  SELECT id, name FROM suppliers ORDER BY name LIMIT 10
) AS top_suppliers;
```

## LIMIT with UNION

Apply LIMIT to the entire result by wrapping in a subquery or appending at the end:

```sql
SELECT name FROM customers
UNION ALL
SELECT name FROM suppliers
LIMIT 20;
```

## Checking Query Plans

Use EXPLAIN to see how MySQL processes your UNION query:

```sql
EXPLAIN
SELECT city FROM customers
UNION ALL
SELECT city FROM suppliers;
```

Look for the `UNION RESULT` row and check whether a temporary table is being created. With `UNION`, a temporary table is always used for deduplication. With `UNION ALL`, MySQL may avoid the temporary table in some cases.

## Summary

`UNION` removes duplicate rows from the combined result and should be used only when uniqueness is required. `UNION ALL` includes all rows from all SELECT statements and is always faster because it skips the deduplication step. When combining data from partitioned tables, log tables, or any non-overlapping sources, always use `UNION ALL` to avoid both incorrect results and unnecessary performance overhead.
