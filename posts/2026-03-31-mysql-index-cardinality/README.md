# How to Check Index Cardinality in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Index, Cardinality, Performance, InnoDB

Description: Learn how to check and interpret index cardinality in MySQL, why it matters for query optimization, and how to refresh stale statistics with ANALYZE TABLE.

---

## What Is Index Cardinality?

Cardinality is the estimated number of distinct values stored in an index. The MySQL query optimizer uses cardinality to decide whether to use an index or perform a full table scan. An index on a column with low cardinality (for example, a boolean `is_active` with 2 distinct values in 10 million rows) may be ignored because a full scan is faster than reading the index and then jumping to each row.

## Viewing Cardinality

```sql
-- From SHOW INDEX
SHOW INDEX FROM orders;

-- The Cardinality column shows estimated distinct values per index
```

```text
+--------+-----------+-------------------+---------+-----------+-------------+
| Table  | Key_name  | Column_name       | Non_uniq| Collation | Cardinality |
+--------+-----------+-------------------+---------+-----------+-------------+
| orders | PRIMARY   | id                |       0 | A         |      500000 |
| orders | idx_cust  | customer_id       |       1 | A         |       12500 |
| orders | idx_status| status            |       1 | A         |           5 |
+--------+-----------+-------------------+---------+-----------+-------------+
```

## Querying from information_schema

```sql
SELECT
    INDEX_NAME,
    COLUMN_NAME,
    SEQ_IN_INDEX,
    CARDINALITY
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'your_database'
  AND TABLE_NAME = 'orders'
ORDER BY INDEX_NAME, SEQ_IN_INDEX;
```

## Calculating Selectivity Percentage

```sql
SELECT
    s.INDEX_NAME,
    s.COLUMN_NAME,
    s.CARDINALITY,
    t.TABLE_ROWS,
    ROUND(s.CARDINALITY / NULLIF(t.TABLE_ROWS, 0) * 100, 2) AS selectivity_pct
FROM information_schema.STATISTICS s
JOIN information_schema.TABLES t
    ON s.TABLE_SCHEMA = t.TABLE_SCHEMA
    AND s.TABLE_NAME = t.TABLE_NAME
WHERE s.TABLE_SCHEMA = 'your_database'
  AND s.TABLE_NAME = 'orders'
  AND s.SEQ_IN_INDEX = 1
ORDER BY selectivity_pct DESC;
```

A selectivity of 100% means every value is unique (like a primary key). Values below 5-10% indicate the optimizer may skip the index.

## When Cardinality Is Stale

InnoDB updates cardinality statistics automatically, but the estimates can drift after large bulk inserts, deletes, or imports. Refresh them manually:

```sql
ANALYZE TABLE orders;
SHOW INDEX FROM orders;  -- now reflects updated estimates
```

`ANALYZE TABLE` is an online operation for InnoDB - it does not block reads or writes.

## Controlling InnoDB Statistics Sampling

The accuracy of cardinality estimates depends on how many pages InnoDB samples:

```sql
-- Per-table sampling pages (higher = more accurate, slower ANALYZE)
ALTER TABLE orders STATS_SAMPLE_PAGES = 50;

-- Global default (MySQL 8.0+)
SET GLOBAL innodb_stats_persistent_sample_pages = 20;
```

## Low Cardinality Columns - What to Do

If a column has inherently low cardinality (like `status` with 5 values), a standalone index may not help. Instead, use it as the second column in a composite index alongside a high-cardinality column:

```sql
-- Better than a standalone idx_status when customer_id is high cardinality
CREATE INDEX idx_customer_status ON orders(customer_id, status);
```

## Summary

Index cardinality is the key metric the MySQL optimizer uses to decide whether to use an index. Check it with `SHOW INDEX` or `information_schema.STATISTICS`, compute selectivity by dividing by table row count, and refresh stale estimates with `ANALYZE TABLE`. For low-cardinality columns, embed them in composite indexes alongside high-cardinality columns rather than indexing them alone.
