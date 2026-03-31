# What Is EXPLAIN in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, EXPLAIN, Query Optimization, Execution Plan, Index

Description: EXPLAIN in MySQL shows the query execution plan, revealing how MySQL accesses tables, which indexes it uses, and how many rows it estimates to examine.

---

## Overview

`EXPLAIN` is a MySQL statement that shows the execution plan for a `SELECT`, `INSERT`, `UPDATE`, `DELETE`, or `TABLE` statement without actually executing it. The output describes how MySQL's optimizer plans to access each table in the query: which indexes it will use, how many rows it estimates it will examine, what join type it will use, and what extra operations (like sorting or temporary tables) will be needed. Understanding `EXPLAIN` output is fundamental to MySQL query optimization.

## Basic Usage

```sql
EXPLAIN SELECT * FROM orders
WHERE customer_id = 42
AND status = 'shipped';
```

For more readable output, use `\G` in the MySQL client:

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42\G
```

## EXPLAIN Output Columns

| Column | Description |
|---|---|
| `id` | Query identifier (higher ID evaluated first for subqueries) |
| `select_type` | Type: SIMPLE, PRIMARY, SUBQUERY, DERIVED, UNION |
| `table` | Which table this row refers to |
| `partitions` | Which partitions will be accessed |
| `type` | Join type (see below) -- most important column |
| `possible_keys` | Indexes MySQL could use |
| `key` | Index MySQL actually chose |
| `key_len` | Length of the chosen key in bytes |
| `ref` | Which columns or constants are used with the key |
| `rows` | Estimated number of rows to examine |
| `filtered` | Estimated percentage of rows remaining after filtering |
| `Extra` | Additional information (filesort, temporary, etc.) |

## The type Column

The `type` column shows the join/access type, from best to worst:

- `system`: Table has only one row.
- `const`: At most one matching row (PRIMARY KEY or UNIQUE lookup).
- `eq_ref`: One row per row from previous table (JOIN on PRIMARY KEY).
- `ref`: Multiple rows match a non-unique index lookup.
- `range`: Index range scan (BETWEEN, IN, >, <).
- `index`: Full index scan.
- `ALL`: Full table scan. This should be investigated for large tables.

## Reading a Simple EXPLAIN

```sql
EXPLAIN SELECT * FROM orders WHERE id = 101;
```

```
id: 1, select_type: SIMPLE, table: orders,
type: const, possible_keys: PRIMARY, key: PRIMARY,
key_len: 4, ref: const, rows: 1, filtered: 100.00, Extra: NULL
```

`type: const` and `key: PRIMARY` show a single-row primary key lookup -- optimal.

## Diagnosing a Full Table Scan

```sql
EXPLAIN SELECT * FROM orders WHERE YEAR(created_at) = 2025;
```

```
type: ALL, key: NULL, rows: 500000, Extra: Using where
```

`type: ALL` with no key and 500,000 rows means a full table scan. Fix by avoiding function wrapping of indexed columns:

```sql
-- Rewrite to allow index use
EXPLAIN SELECT * FROM orders
WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01';
```

## Reading the Extra Column

Common `Extra` values:

- `Using index`: Covered index -- data comes from index only, no table row fetch.
- `Using where`: Filter applied after row fetch.
- `Using filesort`: Sorting done in memory or disk, not using an index.
- `Using temporary`: Intermediate temporary table created (often for GROUP BY/UNION).

```sql
-- "Using temporary; Using filesort" is a warning sign
EXPLAIN SELECT department, COUNT(*) FROM employees
GROUP BY department ORDER BY COUNT(*) DESC;
```

## EXPLAIN for DML

`EXPLAIN` also works for `UPDATE` and `DELETE`:

```sql
EXPLAIN UPDATE orders SET status = 'cancelled'
WHERE customer_id = 99 AND status = 'pending';
```

## EXPLAIN Formats

```sql
-- Traditional tabular format (default)
EXPLAIN SELECT ...;

-- JSON format with more detail
EXPLAIN FORMAT=JSON SELECT ...;

-- Tree format (MySQL 8.0, very readable)
EXPLAIN FORMAT=TREE SELECT ...;
```

## Summary

`EXPLAIN` is the essential first tool when investigating slow MySQL queries. The `type` column reveals how tables are accessed (full scan vs index lookup), `key` shows which index was chosen, and `rows` estimates the work done. `Extra` reveals expensive operations like filesort and temporary tables. Use `EXPLAIN FORMAT=TREE` in MySQL 8.0 for the most readable output, and combine it with `EXPLAIN ANALYZE` to see actual execution statistics.
