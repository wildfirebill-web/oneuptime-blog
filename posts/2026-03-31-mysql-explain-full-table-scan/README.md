# How to Identify Full Table Scans Using EXPLAIN in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, EXPLAIN, Index, Performance, Query

Description: Learn how to use MySQL EXPLAIN to detect full table scans, understand the type column values, and add indexes to eliminate expensive scans.

---

## What Is a Full Table Scan?

A full table scan occurs when MySQL reads every row in a table to satisfy a query rather than using an index. On small tables this is fine, but on tables with millions of rows it becomes a serious performance bottleneck.

MySQL's `EXPLAIN` statement reveals whether a full table scan is happening by showing `ALL` in the `type` column.

## Using EXPLAIN to Detect Full Table Scans

```sql
EXPLAIN SELECT * FROM orders WHERE customer_name = 'Alice';
```

Sample output (abbreviated):

```text
id | select_type | table  | type | key  | rows    | Extra
1  | SIMPLE      | orders | ALL  | NULL | 1500000 | Using where
```

The `type: ALL` value combined with a high `rows` estimate is the definitive indicator of a full table scan.

## Understanding the type Column

The `type` column shows the join or access type from best to worst:

```text
system    - Single row table (constant)
const     - Primary key or unique index lookup
eq_ref    - Unique index join
ref       - Non-unique index lookup
range     - Index range scan
index     - Full index scan (better than ALL)
ALL       - Full table scan (worst)
```

Any query showing `ALL` is a candidate for optimization.

## Identifying the Problem

```sql
-- Example: query with no index on email column
EXPLAIN SELECT id, name FROM users WHERE email = 'bob@example.com';

-- Output shows:
-- type: ALL
-- key: NULL
-- rows: 2000000
-- Extra: Using where
```

The `key: NULL` confirms that no index was used. The `rows` value estimates that MySQL examined 2 million rows to find the matching records.

## Adding an Index to Fix the Full Table Scan

```sql
-- Add an index on the column used in the WHERE clause
CREATE INDEX idx_email ON users(email);

-- Re-run EXPLAIN
EXPLAIN SELECT id, name FROM users WHERE email = 'bob@example.com';

-- Output now shows:
-- type: ref
-- key: idx_email
-- rows: 1
-- Extra: NULL
```

The scan type changes from `ALL` to `ref`, and the rows estimate drops dramatically.

## Multi-Column Conditions

When a query filters on multiple columns, a composite index can eliminate the table scan entirely:

```sql
-- Query filtering on two columns
EXPLAIN SELECT * FROM orders WHERE status = 'pending' AND region = 'west';

-- Add composite index (order matters - most selective first)
CREATE INDEX idx_status_region ON orders(status, region);

-- Re-run EXPLAIN to verify
EXPLAIN SELECT * FROM orders WHERE status = 'pending' AND region = 'west';
-- type should change from ALL to ref
```

## When a Full Table Scan Is Acceptable

MySQL's optimizer may choose a full table scan even when an index exists:

- When the query returns more than roughly 30% of the table rows
- When the table is very small (a few hundred rows)
- When statistics are stale - run `ANALYZE TABLE orders;` to update them

```sql
-- Force index use to compare plans
EXPLAIN SELECT * FROM orders FORCE INDEX (idx_status) WHERE status = 'pending';
```

## Summary

Use `EXPLAIN` and look for `type: ALL` to identify full table scans in MySQL. Once identified, adding an index on the column(s) in the WHERE clause converts the scan to a `ref` or `range` access. Always validate your fix by re-running EXPLAIN after adding the index. On large production tables, even a single unindexed query can lock up a server under load.
