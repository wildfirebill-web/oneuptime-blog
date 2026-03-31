# How to Query INFORMATION_SCHEMA.TABLES in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, INFORMATION_SCHEMA, Metadata, Table, Schema Management

Description: Learn how to query INFORMATION_SCHEMA.TABLES in MySQL to retrieve table metadata including row counts, storage sizes, engine types, and creation dates.

---

## Overview

`INFORMATION_SCHEMA.TABLES` is a metadata catalog that stores information about every table (and view) in your MySQL server. It is invaluable for schema auditing, storage planning, and automating database management tasks.

## Basic Table Listing

```sql
SELECT
  TABLE_SCHEMA,
  TABLE_NAME,
  TABLE_TYPE,
  ENGINE,
  TABLE_ROWS,
  CREATE_TIME
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'myapp'
ORDER BY TABLE_NAME;
```

## Key Columns

| Column | Description |
|--------|-------------|
| `TABLE_SCHEMA` | Database name |
| `TABLE_NAME` | Table name |
| `TABLE_TYPE` | BASE TABLE, VIEW, or SYSTEM VIEW |
| `ENGINE` | Storage engine (InnoDB, MyISAM, etc.) |
| `TABLE_ROWS` | Estimated row count (approximate for InnoDB) |
| `DATA_LENGTH` | Data file size in bytes |
| `INDEX_LENGTH` | Index size in bytes |
| `DATA_FREE` | Free space in the table file |
| `AUTO_INCREMENT` | Next AUTO_INCREMENT value |
| `CREATE_TIME` | Table creation timestamp |
| `UPDATE_TIME` | Last DML operation time |
| `TABLE_COLLATION` | Default character collation |

## Finding the Largest Tables

```sql
SELECT
  TABLE_SCHEMA,
  TABLE_NAME,
  ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_mb,
  ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_mb,
  ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS index_mb,
  TABLE_ROWS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('mysql', 'sys', 'performance_schema', 'information_schema')
ORDER BY total_mb DESC
LIMIT 10;
```

## Finding Tables with High Index-to-Data Ratios

```sql
SELECT
  TABLE_SCHEMA,
  TABLE_NAME,
  ROUND(DATA_LENGTH / 1024 / 1024, 1) AS data_mb,
  ROUND(INDEX_LENGTH / 1024 / 1024, 1) AS index_mb,
  ROUND(INDEX_LENGTH / DATA_LENGTH, 2) AS index_ratio
FROM information_schema.TABLES
WHERE DATA_LENGTH > 0
  AND TABLE_TYPE = 'BASE TABLE'
ORDER BY index_ratio DESC
LIMIT 10;
```

## Listing Tables with Non-InnoDB Engines

```sql
SELECT TABLE_SCHEMA, TABLE_NAME, ENGINE
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('mysql', 'sys', 'performance_schema', 'information_schema')
  AND ENGINE != 'InnoDB'
  AND TABLE_TYPE = 'BASE TABLE';
```

## Finding Tables Without AUTO_INCREMENT

```sql
SELECT TABLE_SCHEMA, TABLE_NAME
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'myapp'
  AND TABLE_TYPE = 'BASE TABLE'
  AND AUTO_INCREMENT IS NULL;
```

## Generating Total Database Size

```sql
SELECT
  TABLE_SCHEMA AS db,
  COUNT(*) AS table_count,
  ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('mysql', 'sys', 'performance_schema', 'information_schema')
GROUP BY TABLE_SCHEMA
ORDER BY total_mb DESC;
```

## Summary

`INFORMATION_SCHEMA.TABLES` is the foundation of MySQL schema introspection. By querying it, you can quickly assess database storage footprints, identify non-standard engine usage, find large tables with fragmentation, and build schema documentation tools. The row count estimates are approximate for InnoDB tables - use `SELECT COUNT(*)` for exact counts when precision is needed.
