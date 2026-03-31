# How to Optimize DISTINCT Queries in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Query Optimization, Distinct, Performance, Index

Description: Learn how to optimize DISTINCT queries in MySQL using indexes, query rewrites, and execution plan analysis to reduce scan overhead.

---

## Overview

`DISTINCT` eliminates duplicate rows from query results. While convenient, DISTINCT queries can be slow because MySQL must sort or hash all matching rows to remove duplicates. Understanding how MySQL executes DISTINCT and applying the right optimizations can dramatically improve performance.

## How MySQL Executes DISTINCT

MySQL processes DISTINCT by:
1. Retrieving all matching rows
2. Sorting or hashing them
3. Removing duplicates

When EXPLAIN shows `Using filesort` or `Using temporary` alongside a DISTINCT query, MySQL is performing an extra deduplication pass.

```sql
EXPLAIN SELECT DISTINCT customer_id FROM orders;
```

## Optimization 1: Use Indexes to Avoid Sorting

If the column(s) in DISTINCT are covered by an index, MySQL can read unique values directly from the index without sorting:

```sql
-- Without index: full table scan + filesort
SELECT DISTINCT customer_id FROM orders;

-- With an index on customer_id: index scan, no sort needed
CREATE INDEX idx_customer_id ON orders (customer_id);
```

Check the execution plan after adding the index:

```sql
EXPLAIN SELECT DISTINCT customer_id FROM orders;
```

Look for `Using index` in the Extra column - this means no table access is needed.

## Optimization 2: Rewrite DISTINCT as GROUP BY

`GROUP BY` can use indexes more efficiently in some cases:

```sql
-- Original
SELECT DISTINCT customer_id FROM orders;

-- Rewrite with GROUP BY
SELECT customer_id FROM orders GROUP BY customer_id;
```

In MySQL, these are often equivalent, but GROUP BY can leverage index-based grouping and avoid sorting for simple cases.

## Optimization 3: Use EXISTS Instead of DISTINCT with JOINs

DISTINCT with JOINs is a common performance trap:

```sql
-- Slow: retrieves all rows then deduplicates
SELECT DISTINCT c.id, c.name
FROM customers c
JOIN orders o ON c.id = o.customer_id;

-- Faster: short-circuits after finding the first match
SELECT id, name
FROM customers c
WHERE EXISTS (
  SELECT 1 FROM orders WHERE customer_id = c.id
);
```

The EXISTS approach stops scanning after finding the first matching order row, while the JOIN approach processes all matching rows.

## Optimization 4: Reduce Column Count in DISTINCT

DISTINCT on fewer columns requires less memory and sorting:

```sql
-- Less efficient: distinct on many columns
SELECT DISTINCT customer_id, status, created_at FROM orders;

-- More efficient: distinct on fewer columns
SELECT DISTINCT customer_id FROM orders;
```

If you need additional columns, fetch them after getting the distinct IDs:

```sql
SELECT c.name, c.email
FROM customers c
WHERE c.id IN (
  SELECT DISTINCT customer_id FROM orders WHERE status = 'completed'
);
```

## Optimization 5: Covering Index for DISTINCT

A covering index that includes all columns in the DISTINCT clause avoids table row lookups entirely:

```sql
CREATE INDEX idx_covering ON orders (customer_id, status);

SELECT DISTINCT customer_id, status FROM orders;
```

EXPLAIN should show `Using index` and no `Using filesort`.

## Checking the Execution Plan

Use EXPLAIN to analyze DISTINCT query performance:

```sql
EXPLAIN SELECT DISTINCT customer_id FROM orders WHERE status = 'pending';
```

Warning signs:
- `Using filesort` - add an index on the DISTINCT column
- `Using temporary` - rewrite with EXISTS or GROUP BY
- `type: ALL` - add a WHERE clause or index

## Summary

Optimize DISTINCT queries in MySQL by adding indexes on the DISTINCT columns, rewriting JOIN-based DISTINCT queries with EXISTS, and using covering indexes. GROUP BY can sometimes replace DISTINCT with better index usage. Always verify improvements with EXPLAIN before and after changes.
