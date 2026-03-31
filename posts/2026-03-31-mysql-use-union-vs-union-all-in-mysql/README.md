# How to Use UNION vs UNION ALL in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Union, Union All, Set Operations, Sql

Description: Understand the differences between UNION and UNION ALL in MySQL, when to use each, and how they affect query performance and results.

---

## Introduction

MySQL provides two set operators for combining SELECT results: `UNION` and `UNION ALL`. The key difference is how they handle duplicate rows. Choosing the right one impacts both the correctness of your results and query performance.

## Key Differences

| Feature | UNION | UNION ALL |
|---------|-------|-----------|
| Removes duplicates | Yes | No |
| Speed | Slower | Faster |
| Extra step | Deduplication | None |
| Memory usage | Higher | Lower |
| Use when | Unique rows needed | All rows needed or no duplicates |

## Basic Syntax Comparison

```sql
-- UNION: removes duplicates
SELECT col FROM table_a
UNION
SELECT col FROM table_b;

-- UNION ALL: keeps all rows
SELECT col FROM table_a
UNION ALL
SELECT col FROM table_b;
```

## Demonstration with Sample Data

```sql
-- table_a contains: 1, 2, 3
-- table_b contains: 2, 3, 4

SELECT val FROM table_a
UNION
SELECT val FROM table_b;
-- Result: 1, 2, 3, 4  (duplicates 2, 3 removed)

SELECT val FROM table_a
UNION ALL
SELECT val FROM table_b;
-- Result: 1, 2, 3, 2, 3, 4  (all 6 rows kept)
```

## When to Use UNION (with Deduplication)

Use `UNION` when combining data from multiple sources where the same row might appear in both, and you want a unique set.

```sql
-- Get all unique email addresses from subscribers and newsletter list
SELECT email FROM subscribers
UNION
SELECT email FROM newsletter_list;

-- Find all unique products ever ordered by a customer
SELECT product_id FROM order_history WHERE customer_id = 100
UNION
SELECT product_id FROM wishlist WHERE customer_id = 100;
```

## When to Use UNION ALL (No Deduplication)

Use `UNION ALL` when:
- Data comes from non-overlapping sources (no duplicates possible).
- You want to count or aggregate all occurrences.
- Performance is a priority and duplicates are acceptable.

```sql
-- Combine log entries from different servers (no dedup needed)
SELECT log_id, message, server_id FROM server1_logs
UNION ALL
SELECT log_id, message, server_id FROM server2_logs;

-- Count all transactions across partitioned tables
SELECT COUNT(*) FROM (
  SELECT id FROM transactions_jan
  UNION ALL
  SELECT id FROM transactions_feb
  UNION ALL
  SELECT id FROM transactions_mar
) AS all_txns;
```

## Performance Comparison

`UNION` performs an additional deduplication step using a sort or hash:

```sql
EXPLAIN
SELECT name FROM list1
UNION
SELECT name FROM list2;
-- Extra: Using temporary  <-- dedup overhead
```

`UNION ALL` has no such overhead:

```sql
EXPLAIN
SELECT name FROM list1
UNION ALL
SELECT name FROM list2;
-- No "Using temporary" for the dedup step
```

## How UNION Removes Duplicates

`UNION` considers rows duplicates if ALL columns in the combined result match. NULL values are treated as equal to each other for deduplication purposes.

```sql
SELECT NULL AS val UNION SELECT NULL AS val;
-- Result: one NULL row (duplicates removed)
```

## ORDER BY Applies to the Whole Result

`ORDER BY` and `LIMIT` at the end of a UNION or UNION ALL apply to the combined result:

```sql
SELECT id, name, 'A' AS source FROM table_a
UNION ALL
SELECT id, name, 'B' AS source FROM table_b
ORDER BY name ASC
LIMIT 20;
```

## Combining Both in a Subquery

You can mix UNION and UNION ALL in nested queries:

```sql
SELECT DISTINCT email FROM (
  SELECT email FROM list_a
  UNION ALL
  SELECT email FROM list_b
  UNION ALL
  SELECT email FROM list_c
) AS combined;
```

This uses `UNION ALL` for speed in combining, then `DISTINCT` to deduplicate.

## Choosing the Right Operator

- If the source tables have no overlapping rows - use `UNION ALL`.
- If you need uniqueness guaranteed - use `UNION`.
- If you are summing or counting all values - use `UNION ALL`.
- If you are building a list for display where duplicates are confusing - use `UNION`.

## Summary

`UNION` removes duplicate rows at the cost of extra processing time, while `UNION ALL` includes all rows and is faster. Use `UNION` when uniqueness is required and `UNION ALL` when you know data will not overlap or when performance matters more than deduplication.
