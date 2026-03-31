# How to Find Redundant Indexes with sys Schema in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, sys Schema, Index, Optimization, Schema Management

Description: Use the MySQL sys schema schema_redundant_indexes view to find duplicate and overlapping indexes that waste storage and slow down writes.

---

## Overview

Redundant indexes occur when one index is a prefix or duplicate of another. For example, if you have indexes on `(a)` and `(a, b)`, the single-column index is redundant because any query using `(a)` alone can use the composite index instead. MySQL's `sys` schema includes the `schema_redundant_indexes` view to find these automatically.

## Using schema_redundant_indexes

```sql
SELECT
  table_schema,
  table_name,
  redundant_index_name,
  redundant_index_columns,
  dominant_index_name,
  dominant_index_columns,
  subpart_exists
FROM sys.schema_redundant_indexes
ORDER BY table_schema, table_name;
```

## Understanding the Output Columns

- `redundant_index_name` - the index that can be dropped
- `dominant_index_name` - the superior index that makes the redundant one unnecessary
- `redundant_index_columns` - columns of the redundant index
- `dominant_index_columns` - columns of the dominant index
- `subpart_exists` - whether the redundant index uses a column prefix

## Example Scenario

```sql
-- Creating a table with redundant indexes
CREATE TABLE orders (
  id INT PRIMARY KEY,
  customer_id INT,
  status VARCHAR(20),
  created_at DATETIME,
  INDEX idx_customer (customer_id),
  INDEX idx_customer_status (customer_id, status),
  INDEX idx_status (status)
);

-- idx_customer is redundant because idx_customer_status starts with customer_id
SELECT redundant_index_name, dominant_index_name
FROM sys.schema_redundant_indexes
WHERE table_name = 'orders';
```

## Generating Drop Statements

```sql
SELECT
  CONCAT(
    'ALTER TABLE `', table_schema, '`.`', table_name,
    '` DROP INDEX `', redundant_index_name, '`; -- replaced by: ',
    dominant_index_name
  ) AS drop_statement
FROM sys.schema_redundant_indexes
WHERE table_schema NOT IN ('mysql', 'sys', 'performance_schema', 'information_schema');
```

## Querying the Underlying View

The `schema_redundant_indexes` view relies on `INFORMATION_SCHEMA.STATISTICS`:

```sql
SELECT
  TABLE_SCHEMA,
  TABLE_NAME,
  INDEX_NAME,
  GROUP_CONCAT(COLUMN_NAME ORDER BY SEQ_IN_INDEX) AS columns
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'myapp'
  AND NON_UNIQUE = 1
GROUP BY TABLE_SCHEMA, TABLE_NAME, INDEX_NAME
ORDER BY TABLE_NAME, columns;
```

## Safe Removal Workflow

1. Run `schema_redundant_indexes` and review each result
2. Verify the dominant index actually covers all queries using the redundant index
3. Use EXPLAIN to confirm query plans remain optimal after dropping
4. Drop in a test environment first
5. Monitor query performance for 24-48 hours after dropping in production

```sql
-- Example drop
ALTER TABLE orders DROP INDEX idx_customer;
```

## Summary

The `sys.schema_redundant_indexes` view identifies index pairs where one index is made unnecessary by another. Removing redundant indexes reduces write amplification, speeds up DML operations, and reduces InnoDB buffer pool pressure. Always validate that the dominant index handles all query patterns before dropping the redundant one.
