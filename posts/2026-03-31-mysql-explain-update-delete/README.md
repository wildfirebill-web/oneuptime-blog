# How to Use EXPLAIN for UPDATE and DELETE Queries in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, EXPLAIN, Query, Performance, Index

Description: Learn how to use MySQL EXPLAIN with UPDATE and DELETE statements to identify full table scans and optimize write query performance.

---

## EXPLAIN Is Not Just for SELECT

Most developers use `EXPLAIN` only with `SELECT` queries, but MySQL 5.6 and later support `EXPLAIN` with `UPDATE`, `DELETE`, and `INSERT ... SELECT` statements as well. This is valuable because slow writes often hold row locks longer, causing cascading slowdowns across your application.

## Using EXPLAIN with UPDATE

```sql
EXPLAIN UPDATE orders SET status = 'shipped' WHERE customer_id = 42;
```

```text
id | select_type | table  | type | key         | rows | Extra
1  | UPDATE      | orders | ref  | idx_customer | 5   | Using where
```

This shows MySQL is using an index (`idx_customer`) to find the rows to update, touching only 5 rows.

## Detecting a Problematic UPDATE

```sql
EXPLAIN UPDATE products SET price = price * 1.1 WHERE category = 'electronics';
```

```text
id | select_type | table    | type | key  | rows   | Extra
1  | UPDATE      | products | ALL  | NULL | 250000 | Using where
```

`type: ALL` on an UPDATE means MySQL scans every row before applying the update. On a busy table this locks rows for the duration of the scan.

```sql
-- Fix: add an index on the filter column
CREATE INDEX idx_category ON products(category);

EXPLAIN UPDATE products SET price = price * 1.1 WHERE category = 'electronics';
-- type: ref, key: idx_category, rows: 8500
```

## Using EXPLAIN with DELETE

```sql
EXPLAIN DELETE FROM logs WHERE created_at < '2025-01-01';
```

```text
id | select_type | table | type  | key         | rows   | Extra
1  | DELETE      | logs  | range | idx_created | 120000 | Using where
```

`type: range` is acceptable here - MySQL is using the index to find a range of rows. Much better than `ALL`.

## Problematic DELETE Example

```sql
-- DELETE with a function in WHERE bypasses the index
EXPLAIN DELETE FROM sessions WHERE YEAR(last_active) = 2024;
```

```text
type: ALL, key: NULL, rows: 2000000
```

The function wrapping `last_active` prevents index use. Rewrite to use a range:

```sql
EXPLAIN DELETE FROM sessions
WHERE last_active >= '2024-01-01' AND last_active < '2025-01-01';
-- type: range, key: idx_last_active
```

## Multi-Table UPDATE with EXPLAIN

```sql
EXPLAIN UPDATE orders o
JOIN customers c ON o.customer_id = c.id
SET o.region = c.region
WHERE c.country = 'US';
```

The output will show rows for each table in the join. Check that both tables use indexes:

```text
id | table     | type | key         | rows
1  | c         | ref  | idx_country | 45000
1  | o         | ref  | idx_customer | 3
```

## Key Columns to Watch for Write Queries

```text
type     - ALL means full scan (bad for writes, causes long lock holds)
key      - NULL means no index (must fix)
rows     - High estimate means many rows locked during write
Extra    - "Using where" with type ALL is a red flag
```

## Before Running Bulk Writes

Always EXPLAIN before running large UPDATE or DELETE operations:

```sql
-- Always do this before bulk operations on large tables
EXPLAIN DELETE FROM audit_logs WHERE event_type = 'debug' LIMIT 10000;

-- If rows estimate is in millions, add an index first
CREATE INDEX idx_event_type ON audit_logs(event_type);
```

## Summary

EXPLAIN works with UPDATE and DELETE in MySQL 5.6 and later. Use it before any bulk write operation to detect full table scans that will hold locks on every matching row for the full duration of the operation. The fix is always the same as for SELECT - add an index on the WHERE clause columns. On busy OLTP databases, an unindexed DELETE or UPDATE can bring your application to a halt.
