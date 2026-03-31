# How to Optimize IN Clause with Large Lists in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Query, Index, Optimization, Performance

Description: Learn how to optimize MySQL IN clause performance with large value lists, including when to use temporary tables, JOINs, and the eq_range_index_dive_limit setting.

---

## The Problem with Large IN Lists

The `IN` clause is one of the most convenient SQL constructs, but its performance degrades in two distinct ways when the list grows large:

1. The optimizer may underestimate or over-estimate cardinality for large lists, choosing a suboptimal plan
2. The query text itself becomes enormous, increasing parse time and memory usage

## Small IN Lists (Efficient)

For small lists (under a few hundred values), `IN` with an indexed column performs well:

```sql
-- Index on status column
EXPLAIN SELECT * FROM orders WHERE status IN ('pending', 'processing', 'shipped');
```

```text
type: range, key: idx_status, rows: 5000
```

MySQL uses a range scan over the index for each value in the list.

## Large IN Lists and eq_range_index_dive_limit

When IN lists exceed the `eq_range_index_dive_limit` threshold (default: 200 in MySQL 8.0), the optimizer stops doing individual cardinality lookups per value and uses a single estimate. This can cause poor plan choices.

```sql
-- Check current limit
SHOW VARIABLES LIKE 'eq_range_index_dive_limit';

-- Increase if your queries use large IN lists with good selectivity
SET SESSION eq_range_index_dive_limit = 500;
```

## Rewrite Large IN as a JOIN with a Temporary Table

When the IN list comes from application code and has hundreds or thousands of values, loading them into a temporary table and joining is more efficient:

```sql
-- Instead of this (hundreds of values)
SELECT * FROM products WHERE id IN (101, 102, 103, ... 5000);

-- Create a temporary table
CREATE TEMPORARY TABLE tmp_ids (id INT PRIMARY KEY);
INSERT INTO tmp_ids VALUES (101), (102), (103), ...;

-- JOIN instead of IN
SELECT p.*
FROM products p
JOIN tmp_ids t ON p.id = t.id;
```

The JOIN uses the primary key of `products` for each lookup, and the primary key of `tmp_ids` for driving the join.

## IN with Subquery vs Explicit List

When the IN list comes from another table, always use a subquery or JOIN rather than materializing the list in application code:

```sql
-- Application-generated list (bad for large sets)
SELECT * FROM orders WHERE customer_id IN (1001, 1002, 1003, ..., 50000);

-- Subquery approach (let MySQL optimize it)
SELECT * FROM orders
WHERE customer_id IN (SELECT id FROM customers WHERE tier = 'premium');

-- Or explicit JOIN
SELECT o.*
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.tier = 'premium';
```

## Partitioning IN Lists

If you must use large IN lists, splitting them into smaller batches often improves total performance:

```python
# Application-level batching example
batch_size = 200
for i in range(0, len(ids), batch_size):
    batch = ids[i:i + batch_size]
    placeholders = ','.join(['%s'] * len(batch))
    cursor.execute(f"SELECT * FROM orders WHERE id IN ({placeholders})", batch)
```

## NULL Handling in IN

Remember that `IN` does not match NULL values:

```sql
-- This will NOT return rows where region IS NULL
SELECT * FROM customers WHERE region NOT IN ('west', 'east');

-- Must explicitly handle NULL
SELECT * FROM customers WHERE region NOT IN ('west', 'east') OR region IS NULL;
```

## EXPLAIN Check for IN Clause Plans

```sql
EXPLAIN SELECT * FROM products WHERE category_id IN (1, 2, 3, 4, 5);
-- Look for: type: range, key used
-- If type: ALL, the column is not indexed

EXPLAIN SELECT * FROM products WHERE category_id IN (SELECT id FROM categories WHERE active = 1);
-- In MySQL 8.0, may show SEMI JOIN in SHOW WARNINGS output
```

## Summary

MySQL handles small IN lists efficiently using range index scans. As lists grow beyond 200 values, optimizer cost estimates become less accurate and query parse overhead increases. The best solutions for large lists are: join to a temporary table containing the IDs, use a subquery that MySQL can convert to a semijoin, or batch the IN list in application code. Always verify the execution plan with EXPLAIN when using IN with more than a few dozen values.
