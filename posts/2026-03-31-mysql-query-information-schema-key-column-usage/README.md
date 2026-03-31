# How to Query INFORMATION_SCHEMA.KEY_COLUMN_USAGE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, INFORMATION_SCHEMA, Foreign Key, Constraint, Schema

Description: Learn how to query INFORMATION_SCHEMA.KEY_COLUMN_USAGE in MySQL to inspect primary key, unique key, and foreign key column mappings.

---

## Overview

`INFORMATION_SCHEMA.KEY_COLUMN_USAGE` provides information about columns that participate in key constraints - primary keys, unique keys, and foreign keys. It maps constraint names to their constituent columns and, for foreign keys, shows the referenced table and column.

## Basic Query

```sql
SELECT
  CONSTRAINT_NAME,
  TABLE_NAME,
  COLUMN_NAME,
  ORDINAL_POSITION,
  POSITION_IN_UNIQUE_CONSTRAINT,
  REFERENCED_TABLE_NAME,
  REFERENCED_COLUMN_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = 'myapp'
  AND TABLE_NAME = 'orders'
ORDER BY CONSTRAINT_NAME, ORDINAL_POSITION;
```

## Finding All Foreign Keys in a Schema

```sql
SELECT
  TABLE_NAME,
  CONSTRAINT_NAME,
  COLUMN_NAME,
  REFERENCED_TABLE_NAME,
  REFERENCED_COLUMN_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = 'myapp'
  AND REFERENCED_TABLE_NAME IS NOT NULL
ORDER BY TABLE_NAME, CONSTRAINT_NAME;
```

## Mapping Parent-Child Relationships

```sql
SELECT
  CONCAT(TABLE_NAME, '.', COLUMN_NAME) AS child_col,
  CONCAT(REFERENCED_TABLE_NAME, '.', REFERENCED_COLUMN_NAME) AS parent_col,
  CONSTRAINT_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = 'myapp'
  AND REFERENCED_TABLE_NAME IS NOT NULL
ORDER BY TABLE_NAME;
```

## Finding Tables That Reference a Specific Table

Useful before dropping a table to identify dependencies:

```sql
SELECT
  TABLE_NAME AS dependent_table,
  CONSTRAINT_NAME,
  COLUMN_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = 'myapp'
  AND REFERENCED_TABLE_NAME = 'customers';
```

## Finding Composite Foreign Keys

```sql
SELECT
  TABLE_NAME,
  CONSTRAINT_NAME,
  GROUP_CONCAT(COLUMN_NAME ORDER BY ORDINAL_POSITION) AS fk_columns,
  REFERENCED_TABLE_NAME,
  GROUP_CONCAT(REFERENCED_COLUMN_NAME ORDER BY ORDINAL_POSITION) AS ref_columns
FROM information_schema.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = 'myapp'
  AND REFERENCED_TABLE_NAME IS NOT NULL
GROUP BY TABLE_NAME, CONSTRAINT_NAME, REFERENCED_TABLE_NAME
HAVING COUNT(*) > 1;
```

## Combining with TABLE_CONSTRAINTS

```sql
SELECT
  tc.CONSTRAINT_TYPE,
  kcu.TABLE_NAME,
  kcu.CONSTRAINT_NAME,
  kcu.COLUMN_NAME,
  kcu.REFERENCED_TABLE_NAME
FROM information_schema.TABLE_CONSTRAINTS tc
JOIN information_schema.KEY_COLUMN_USAGE kcu
  ON tc.CONSTRAINT_NAME = kcu.CONSTRAINT_NAME
  AND tc.TABLE_SCHEMA = kcu.TABLE_SCHEMA
  AND tc.TABLE_NAME = kcu.TABLE_NAME
WHERE tc.TABLE_SCHEMA = 'myapp'
ORDER BY tc.CONSTRAINT_TYPE, kcu.TABLE_NAME;
```

## Finding FK Columns Not Indexed on the Child Side

Foreign key columns should be indexed to avoid full table scans during cascade operations:

```sql
SELECT
  kcu.TABLE_NAME,
  kcu.COLUMN_NAME,
  kcu.CONSTRAINT_NAME
FROM information_schema.KEY_COLUMN_USAGE kcu
LEFT JOIN information_schema.STATISTICS s
  ON kcu.TABLE_SCHEMA = s.TABLE_SCHEMA
  AND kcu.TABLE_NAME = s.TABLE_NAME
  AND kcu.COLUMN_NAME = s.COLUMN_NAME
WHERE kcu.TABLE_SCHEMA = 'myapp'
  AND kcu.REFERENCED_TABLE_NAME IS NOT NULL
  AND s.COLUMN_NAME IS NULL;
```

## Summary

`INFORMATION_SCHEMA.KEY_COLUMN_USAGE` is the definitive source for understanding key constraint column mappings in MySQL. By querying it, you can map entity relationships, detect unindexed foreign key columns, identify tables with complex composite keys, and safely plan schema changes that might affect referential integrity.
