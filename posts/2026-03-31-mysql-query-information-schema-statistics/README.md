# How to Query INFORMATION_SCHEMA.STATISTICS (Indexes) in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, INFORMATION_SCHEMA, Index, Statistics, Schema Management

Description: Learn how to query INFORMATION_SCHEMA.STATISTICS in MySQL to inspect all indexes, their columns, cardinality, and uniqueness across your database schema.

---

## Overview

`INFORMATION_SCHEMA.STATISTICS` provides detailed metadata about every index on every table in MySQL. It is the equivalent of `SHOW INDEX FROM table_name` but queryable across all tables and schemas simultaneously, making it perfect for index audits.

## Basic Index Listing

```sql
SELECT
  TABLE_NAME,
  INDEX_NAME,
  SEQ_IN_INDEX,
  COLUMN_NAME,
  NON_UNIQUE,
  CARDINALITY,
  INDEX_TYPE
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'myapp'
  AND TABLE_NAME = 'orders'
ORDER BY INDEX_NAME, SEQ_IN_INDEX;
```

## Key Columns

| Column | Description |
|--------|-------------|
| `TABLE_SCHEMA` | Database name |
| `TABLE_NAME` | Table name |
| `INDEX_NAME` | Index name (PRIMARY for primary key) |
| `SEQ_IN_INDEX` | Column position within the index |
| `COLUMN_NAME` | Column included in the index |
| `NON_UNIQUE` | 0 = unique, 1 = non-unique |
| `CARDINALITY` | Estimated distinct values (affects query planning) |
| `INDEX_TYPE` | BTREE, HASH, FULLTEXT, or SPATIAL |
| `SUB_PART` | Number of chars indexed for prefix indexes |
| `NULLABLE` | YES if column can be NULL |

## Listing All Indexes for a Schema

```sql
SELECT
  TABLE_NAME,
  INDEX_NAME,
  GROUP_CONCAT(COLUMN_NAME ORDER BY SEQ_IN_INDEX SEPARATOR ', ') AS columns,
  NON_UNIQUE,
  INDEX_TYPE
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'myapp'
GROUP BY TABLE_NAME, INDEX_NAME, NON_UNIQUE, INDEX_TYPE
ORDER BY TABLE_NAME, INDEX_NAME;
```

## Finding Tables Without a Primary Key

```sql
SELECT t.TABLE_NAME
FROM information_schema.TABLES t
LEFT JOIN information_schema.STATISTICS s
  ON t.TABLE_SCHEMA = s.TABLE_SCHEMA
  AND t.TABLE_NAME = s.TABLE_NAME
  AND s.INDEX_NAME = 'PRIMARY'
WHERE t.TABLE_SCHEMA = 'myapp'
  AND t.TABLE_TYPE = 'BASE TABLE'
  AND s.INDEX_NAME IS NULL;
```

## Finding Low-Cardinality Indexes

Low cardinality on a non-unique index means the index is unlikely to be selective enough to benefit queries:

```sql
SELECT
  TABLE_NAME,
  INDEX_NAME,
  COLUMN_NAME,
  CARDINALITY
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'myapp'
  AND SEQ_IN_INDEX = 1
  AND INDEX_NAME != 'PRIMARY'
  AND CARDINALITY < 10
ORDER BY CARDINALITY;
```

## Finding Prefix Indexes

```sql
SELECT
  TABLE_NAME,
  INDEX_NAME,
  COLUMN_NAME,
  SUB_PART
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'myapp'
  AND SUB_PART IS NOT NULL;
```

## Counting Indexes per Table

```sql
SELECT
  TABLE_NAME,
  COUNT(DISTINCT INDEX_NAME) AS index_count,
  COUNT(DISTINCT INDEX_NAME) - 1 AS secondary_indexes
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'myapp'
GROUP BY TABLE_NAME
ORDER BY index_count DESC;
```

## Summary

`INFORMATION_SCHEMA.STATISTICS` is the authoritative source for index metadata in MySQL. By querying cardinality, column composition, and index types, you can audit your schema for missing primary keys, low-selectivity indexes, and opportunities to consolidate single-column indexes into composite ones that serve multiple query patterns.
