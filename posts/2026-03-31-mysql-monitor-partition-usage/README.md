# How to Monitor Partition Usage in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Partition, INFORMATION_SCHEMA, Monitoring, Performance

Description: Learn how to monitor MySQL partition usage using INFORMATION_SCHEMA.PARTITIONS to track row counts, data sizes, and identify unbalanced or problematic partitions.

---

## Why Monitor Partition Usage?

Partitioned tables require ongoing attention. Partitions can grow unevenly (hot partitions), a catch-all partition may fill unexpectedly, or individual partitions may become fragmented. The `INFORMATION_SCHEMA.PARTITIONS` view provides real-time statistics on every partition's row count, data size, and fragmentation state.

## Columns in INFORMATION_SCHEMA.PARTITIONS

Key columns for monitoring:

- `TABLE_SCHEMA` - database name
- `TABLE_NAME` - table name
- `PARTITION_NAME` - partition name
- `SUBPARTITION_NAME` - subpartition name (if applicable)
- `PARTITION_ORDINAL_POSITION` - order of the partition
- `PARTITION_METHOD` - RANGE, LIST, HASH, KEY, etc.
- `PARTITION_EXPRESSION` - the partitioning expression
- `PARTITION_DESCRIPTION` - boundary value (for RANGE/LIST)
- `TABLE_ROWS` - estimated row count
- `DATA_LENGTH` - bytes of data
- `INDEX_LENGTH` - bytes of indexes
- `DATA_FREE` - bytes of fragmented free space

## View All Partitions for a Table

```sql
SELECT
    PARTITION_NAME,
    PARTITION_DESCRIPTION,
    TABLE_ROWS,
    ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_mb,
    ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS index_mb,
    ROUND(DATA_FREE / 1024 / 1024, 2) AS free_mb
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'orders'
ORDER BY PARTITION_ORDINAL_POSITION;
```

## Find the Largest Partitions Across All Tables

```sql
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    PARTITION_NAME,
    TABLE_ROWS,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1048576, 2) AS total_mb
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_SCHEMA NOT IN (
    'information_schema', 'performance_schema', 'mysql', 'sys'
)
  AND PARTITION_NAME IS NOT NULL
ORDER BY total_mb DESC
LIMIT 20;
```

## Detect the MAXVALUE Partition Growing

A catch-all `MAXVALUE` partition accumulating rows indicates missing explicit partitions:

```sql
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    PARTITION_NAME,
    PARTITION_DESCRIPTION,
    TABLE_ROWS
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE PARTITION_DESCRIPTION = 'MAXVALUE'
  AND TABLE_ROWS > 100000
ORDER BY TABLE_ROWS DESC;
```

## Identify Unbalanced HASH/KEY Partitions

For HASH-partitioned tables, check distribution evenness:

```sql
SELECT
    PARTITION_NAME,
    TABLE_ROWS,
    ROUND(AVG(TABLE_ROWS) OVER (), 0) AS avg_rows,
    TABLE_ROWS - ROUND(AVG(TABLE_ROWS) OVER (), 0) AS deviation
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'user_events'
ORDER BY deviation DESC;
```

## Find Fragmented Partitions

High `DATA_FREE` signals fragmentation worth reclaiming:

```sql
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    PARTITION_NAME,
    ROUND(DATA_FREE / 1048576, 2) AS free_mb,
    ROUND(DATA_FREE / (DATA_LENGTH + 1) * 100, 1) AS fragmentation_pct
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE DATA_FREE > 10485760
  AND PARTITION_NAME IS NOT NULL
ORDER BY fragmentation_pct DESC;
```

Reclaim fragmented space with:

```sql
ALTER TABLE orders OPTIMIZE PARTITION p2023;
```

## Monitor Empty Partitions

Future partitions that have not yet received data:

```sql
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    PARTITION_NAME,
    PARTITION_DESCRIPTION
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_ROWS = 0
  AND PARTITION_NAME IS NOT NULL
  AND TABLE_SCHEMA NOT IN ('information_schema', 'performance_schema', 'mysql', 'sys')
ORDER BY TABLE_SCHEMA, TABLE_NAME;
```

## Summary

`INFORMATION_SCHEMA.PARTITIONS` is the primary tool for ongoing partition health monitoring. Use it to track data growth per partition, detect catch-all partitions filling up, identify fragmentation, and verify that HASH/KEY distributions remain reasonably balanced. Regular monitoring queries help you stay ahead of partition management tasks before they become performance problems.
