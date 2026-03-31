# How to Analyze Index Usage in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Index, Performance Schema, Optimization, Query

Description: Learn how to analyze MySQL index usage using performance_schema, sys schema views, and EXPLAIN to find unused indexes, verify index selection, and improve queries.

---

## Why Analyze Index Usage?

Every index consumes storage and imposes write overhead on INSERT, UPDATE, and DELETE operations. Analyzing index usage helps you identify indexes the optimizer never selects (candidates for removal) and missing indexes (opportunities to speed up slow queries).

## Using performance_schema to Find Unused Indexes

MySQL tracks index access in the `performance_schema.table_io_waits_summary_by_index_usage` table:

```sql
SELECT
    OBJECT_SCHEMA,
    OBJECT_NAME,
    INDEX_NAME,
    COUNT_READ,
    COUNT_FETCH,
    COUNT_INSERT,
    COUNT_UPDATE,
    COUNT_DELETE
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE OBJECT_SCHEMA = 'your_database'
  AND COUNT_READ = 0
  AND COUNT_FETCH = 0
  AND INDEX_NAME IS NOT NULL
  AND INDEX_NAME != 'PRIMARY'
ORDER BY OBJECT_NAME, INDEX_NAME;
```

Indexes with `COUNT_READ = 0` and `COUNT_FETCH = 0` have not been used since the last server restart or statistics reset.

## Using the sys Schema

The `sys` schema provides a convenient view:

```sql
SELECT object_schema, object_name, index_name
FROM sys.schema_unused_indexes
WHERE object_schema = 'your_database';
```

## Verifying Index Use with EXPLAIN

Before concluding an index is unused, check whether your key queries actually use it:

```sql
EXPLAIN SELECT id, status FROM orders
WHERE customer_id = 1234
  AND status = 'pending'\G
```

Look for the `key` field - it shows which index MySQL chose. If it says `NULL`, no index was used.

## Checking Index Selectivity

Low selectivity indexes may be bypassed by the optimizer:

```sql
SELECT
    INDEX_NAME,
    CARDINALITY,
    (SELECT TABLE_ROWS FROM information_schema.TABLES
     WHERE TABLE_SCHEMA = 'your_database' AND TABLE_NAME = 'orders') AS total_rows,
    ROUND(CARDINALITY /
        (SELECT TABLE_ROWS FROM information_schema.TABLES
         WHERE TABLE_SCHEMA = 'your_database' AND TABLE_NAME = 'orders') * 100, 2
    ) AS selectivity_pct
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'your_database'
  AND TABLE_NAME = 'orders'
  AND SEQ_IN_INDEX = 1
ORDER BY selectivity_pct;
```

Indexes with selectivity below 5% are often skipped by the optimizer in favor of a full table scan.

## Resetting Usage Counters

```sql
-- Reset performance_schema counters
TRUNCATE TABLE performance_schema.table_io_waits_summary_by_index_usage;
```

After resetting, run your workload for a representative period before evaluating again.

## Finding Redundant Indexes

```sql
SELECT
    a.TABLE_NAME,
    a.INDEX_NAME AS index_1,
    b.INDEX_NAME AS index_2,
    a.COLUMN_NAME
FROM information_schema.STATISTICS a
JOIN information_schema.STATISTICS b
    ON a.TABLE_SCHEMA = b.TABLE_SCHEMA
    AND a.TABLE_NAME = b.TABLE_NAME
    AND a.COLUMN_NAME = b.COLUMN_NAME
    AND a.SEQ_IN_INDEX = b.SEQ_IN_INDEX
    AND a.INDEX_NAME < b.INDEX_NAME
WHERE a.TABLE_SCHEMA = 'your_database';
```

## Summary

Analyze index usage with `performance_schema.table_io_waits_summary_by_index_usage` and `sys.schema_unused_indexes` to find indexes that are never read. Verify selectivity with the cardinality query and confirm optimizer choices with `EXPLAIN`. Reset counters after changes, then observe again before making removal decisions.
