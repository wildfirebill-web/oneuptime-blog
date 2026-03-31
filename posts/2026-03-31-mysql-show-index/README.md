# How to Use SHOW INDEX in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Index, Schema, Cardinality, InnoDB

Description: Learn how to use SHOW INDEX in MySQL to retrieve index metadata including cardinality, column order, nullability, index type, and visibility for query tuning.

---

## Basic Syntax

`SHOW INDEX` returns a row for each column included in each index on a table. Composite indexes produce multiple rows - one per column.

```sql
SHOW INDEX FROM orders;
SHOW INDEXES FROM orders;  -- alternate spelling
SHOW KEYS FROM orders;     -- another synonym
```

## Understanding the Output Columns

```text
+---------+------------+--------------+--------------+-------------+-----------+-------------+----------+--------+------+...
| Table   | Non_unique | Key_name     | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null |...
+---------+------------+--------------+--------------+-------------+-----------+-------------+----------+--------+------+...
| orders  |          0 | PRIMARY      |            1 | id          | A         |      500000 | NULL     | NULL   |      |...
| orders  |          1 | idx_customer |            1 | customer_id | A         |       12000 | NULL     | NULL   |      |...
| orders  |          1 | idx_date_st  |            1 | created_at  | A         |      490000 | NULL     | NULL   |      |...
| orders  |          1 | idx_date_st  |            2 | status      | A         |      500000 | NULL     | NULL   |      |...
+---------+------------+--------------+--------------+-------------+-----------+-------------+----------+--------+------+...
```

| Column | Description |
|--------|-------------|
| Non_unique | 0 = unique index, 1 = allows duplicates |
| Key_name | Index name (PRIMARY for primary key) |
| Seq_in_index | Column position in composite index (1-based) |
| Column_name | The indexed column |
| Collation | A = ascending, D = descending, NULL = not sorted |
| Cardinality | Estimated distinct values (used by optimizer) |
| Sub_part | Prefix length if a prefix index, NULL otherwise |
| Null | YES if the column can contain NULL values |
| Index_type | BTREE, HASH, FULLTEXT, or SPATIAL |
| Visible | YES or NO (invisible indexes) |

## Filtering Results

```sql
-- Only unique indexes
SHOW INDEX FROM orders WHERE Non_unique = 0;

-- Only full-text indexes
SHOW INDEX FROM orders WHERE Index_type = 'FULLTEXT';

-- Only prefix indexes
SHOW INDEX FROM orders WHERE Sub_part IS NOT NULL;
```

## Checking Cardinality

Cardinality is an estimate maintained by InnoDB. Low cardinality relative to total rows indicates the optimizer may choose not to use that index.

```sql
-- Compare cardinality to row count
SELECT
    s.INDEX_NAME,
    s.COLUMN_NAME,
    s.CARDINALITY,
    t.TABLE_ROWS,
    ROUND(s.CARDINALITY / t.TABLE_ROWS * 100, 1) AS selectivity_pct
FROM information_schema.STATISTICS s
JOIN information_schema.TABLES t
    ON s.TABLE_SCHEMA = t.TABLE_SCHEMA AND s.TABLE_NAME = t.TABLE_NAME
WHERE s.TABLE_SCHEMA = 'your_database'
  AND s.TABLE_NAME = 'orders'
ORDER BY selectivity_pct DESC;
```

## Refreshing Cardinality Statistics

If cardinality values seem stale or inaccurate after large data changes:

```sql
ANALYZE TABLE orders;
SHOW INDEX FROM orders;  -- now shows updated cardinality
```

## Viewing All Indexes in a Database

```sql
SELECT TABLE_NAME, INDEX_NAME, COLUMN_NAME, CARDINALITY, INDEX_TYPE, IS_VISIBLE
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'your_database'
ORDER BY TABLE_NAME, INDEX_NAME, SEQ_IN_INDEX;
```

## Summary

`SHOW INDEX` is the primary tool for inspecting index metadata in MySQL. Pay attention to `Cardinality` for optimizer selectivity, `Sub_part` for prefix indexes, `Collation` for descending indexes, and `Visible` for invisible indexes. Use `ANALYZE TABLE` to refresh stale cardinality estimates and query `information_schema.STATISTICS` for cross-table analysis.
