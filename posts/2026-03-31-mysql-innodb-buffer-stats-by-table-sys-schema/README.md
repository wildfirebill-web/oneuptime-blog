# How to Use the innodb_buffer_stats_by_table View in MySQL sys Schema

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, sys Schema, InnoDB, Buffer Pool, Performance

Description: Use the MySQL sys schema innodb_buffer_stats_by_table view to see which tables are consuming the most InnoDB buffer pool space.

---

## Overview

The InnoDB buffer pool is MySQL's primary caching mechanism. Understanding which tables occupy the most buffer pool pages helps you tune `innodb_buffer_pool_size`, identify hot tables, and diagnose cache eviction pressure. The `sys.innodb_buffer_stats_by_table` view aggregates this data by table.

## Basic Query

```sql
SELECT *
FROM sys.innodb_buffer_stats_by_table
ORDER BY allocated DESC
LIMIT 10;
```

## Key Columns

```sql
SELECT
  object_schema,
  object_name,
  allocated,
  data,
  pages,
  pages_hashed,
  pages_old,
  rows_cached
FROM sys.innodb_buffer_stats_by_table
ORDER BY pages DESC
LIMIT 15;
```

- `allocated` - total buffer pool memory allocated to this table (formatted)
- `data` - data bytes cached
- `pages` - total buffer pool pages occupied
- `pages_hashed` - pages in the adaptive hash index
- `pages_old` - pages that are in the old half of the LRU list (candidates for eviction)
- `rows_cached` - estimated cached row count

## Finding Tables That Dominate the Buffer Pool

```sql
SELECT
  object_schema,
  object_name,
  pages,
  allocated,
  rows_cached
FROM sys.innodb_buffer_stats_by_table
WHERE object_schema NOT IN ('mysql', 'sys', 'information_schema', 'performance_schema')
ORDER BY pages DESC
LIMIT 10;
```

## Identifying Cold Data Consuming Buffer Pool

A high `pages_old` percentage indicates the table's data is not actively used but occupies valuable buffer pool space:

```sql
SELECT
  object_schema,
  object_name,
  pages,
  pages_old,
  ROUND(pages_old / pages * 100, 1) AS old_pct
FROM sys.innodb_buffer_stats_by_table
WHERE pages > 100
ORDER BY old_pct DESC
LIMIT 10;
```

## Using the x$ Variant

```sql
SELECT
  object_schema,
  object_name,
  allocated / 1024 / 1024 AS allocated_mb,
  pages,
  rows_cached
FROM sys.x$innodb_buffer_stats_by_table
ORDER BY allocated DESC
LIMIT 10;
```

## Querying the Source Data Directly

```sql
SELECT
  TABLE_NAME,
  COUNT(*) AS pages,
  SUM(DATA_SIZE) / 1024 / 1024 AS data_mb,
  SUM(PAGE_TYPE = 'INDEX') AS index_pages
FROM information_schema.INNODB_BUFFER_PAGE
WHERE TABLE_NAME IS NOT NULL
GROUP BY TABLE_NAME
ORDER BY pages DESC
LIMIT 15;
```

## Buffer Pool Schema Breakdown

```sql
SELECT *
FROM sys.innodb_buffer_stats_by_schema
ORDER BY allocated DESC;
```

## Practical Insights

If a single table occupies more than 20-30% of the buffer pool, consider:
1. Increasing `innodb_buffer_pool_size` if memory is available
2. Partitioning the table to separate hot and cold partitions
3. Using buffer pool instances (`innodb_buffer_pool_instances`) if the pool is large
4. Archiving old data out of the table

## Summary

The `sys.innodb_buffer_stats_by_table` view exposes how InnoDB buffer pool space is distributed across tables. By identifying tables that consume the most pages and those with high cold-page ratios, you can make informed decisions about buffer pool sizing, data archival, and partitioning strategies to improve cache hit rates.
