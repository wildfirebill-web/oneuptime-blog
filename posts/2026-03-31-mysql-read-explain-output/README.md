# How to Read EXPLAIN Output in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, EXPLAIN, Query, Optimization, Performance

Description: Learn how to read and interpret MySQL EXPLAIN output, understand each column's meaning, and use the information to diagnose and fix slow queries.

---

## Running EXPLAIN

Prefix any `SELECT`, `INSERT`, `UPDATE`, or `DELETE` with `EXPLAIN` to see the optimizer's execution plan:

```sql
EXPLAIN SELECT o.id, o.total, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'pending'
ORDER BY o.created_at DESC
LIMIT 10\G
```

Add `\G` for vertical formatting, which is easier to read for multi-table queries.

## The Key Columns

### id

The query block number. All rows with the same `id` are part of the same SELECT. Subqueries or UNION steps get higher `id` values.

### select_type

How MySQL classifies this part of the query:

| Value | Meaning |
|-------|---------|
| SIMPLE | No subqueries or UNION |
| PRIMARY | Outer query of a UNION or subquery |
| SUBQUERY | Inner SELECT in a WHERE clause |
| DERIVED | Subquery in the FROM clause |
| UNION | Second or later table in a UNION |

### table

The table or alias being accessed in this step.

### type

The join type - the most important column for performance diagnosis. See the section below.

### possible_keys

Indexes MySQL considered. `NULL` means no index is applicable.

### key

The index MySQL actually chose. `NULL` means a full table scan.

### key_len

The number of bytes used from the index. For composite indexes, a shorter `key_len` means not all columns were used.

### rows

Estimated number of rows MySQL expects to examine for this step. Lower is better.

### filtered

Estimated percentage of rows that pass the `WHERE` filter after the index lookup. `rows * filtered / 100` gives the estimated output rows.

### Extra

Additional information about the execution:

| Value | Meaning |
|-------|---------|
| Using index | Covering index - no row lookup needed |
| Using where | Post-index filter applied |
| Using filesort | Sort cannot be satisfied by index - slow for large sets |
| Using temporary | Temporary table required - watch for GROUP BY without index |
| Using index condition | Index Condition Pushdown (ICP) used |

## The type Column Values (Best to Worst)

```text
system > const > eq_ref > ref > range > index > ALL
```

- `const` / `eq_ref`: at most one row matched - very fast
- `ref`: all rows matching a non-unique index value
- `range`: index range scan (BETWEEN, IN, >, <)
- `index`: full index scan - better than ALL but still slow on large indexes
- `ALL`: full table scan - no index used

## Example: Diagnosing a Bad Plan

```sql
EXPLAIN SELECT * FROM orders WHERE YEAR(created_at) = 2025\G
```

```text
type: ALL
key: NULL
rows: 500000
Extra: Using where
```

`type: ALL` reveals a full scan because `YEAR(created_at)` prevents index use. Fix: add a functional index or rewrite the predicate:

```sql
-- Rewrite to use range (index-friendly)
WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01'
```

```sql
EXPLAIN SELECT * FROM orders
WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01'\G
```

```text
type: range
key: idx_created_at
rows: 42000
Extra: Using index condition
```

## Summary

Read EXPLAIN output by focusing on `type` (aim for `const`, `ref`, or `range`), `key` (should be non-NULL for filtered queries), `rows` (lower is better), and `Extra` (avoid `Using filesort` and `Using temporary` on large sets). Iteratively apply changes and re-run EXPLAIN until the plan is acceptable.
