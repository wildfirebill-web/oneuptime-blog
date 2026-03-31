# How to Find Database Sizes Using INFORMATION_SCHEMA in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, INFORMATION_SCHEMA, Performance, Disk Usage, Database Administration

Description: Learn how to calculate the total size of databases in MySQL using INFORMATION_SCHEMA.TABLES, with queries for data size, index size, and growth tracking.

---

## Why Measure Database Sizes?

Monitoring database sizes is a routine part of capacity management. Knowing which databases consume the most disk space helps you plan storage allocation, set up alerting for runaway growth, and prioritize archiving or purging operations. MySQL does not provide a single command to show all database sizes, but `INFORMATION_SCHEMA.TABLES` makes it straightforward.

## Aggregate Table Sizes by Database

Sum the data and index lengths grouped by schema to get database-level totals:

```sql
SELECT
    TABLE_SCHEMA AS database_name,
    ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_size_mb,
    ROUND(SUM(DATA_LENGTH) / 1024 / 1024, 2) AS data_size_mb,
    ROUND(SUM(INDEX_LENGTH) / 1024 / 1024, 2) AS index_size_mb,
    COUNT(*) AS table_count
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA NOT IN (
    'information_schema', 'performance_schema', 'mysql', 'sys'
)
GROUP BY TABLE_SCHEMA
ORDER BY (SUM(DATA_LENGTH + INDEX_LENGTH)) DESC;
```

## Show Sizes in GB for Large Databases

```sql
SELECT
    TABLE_SCHEMA AS database_name,
    ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1073741824, 3) AS total_gb
FROM INFORMATION_SCHEMA.TABLES
GROUP BY TABLE_SCHEMA
ORDER BY total_gb DESC;
```

## Find a Single Database's Size

```sql
SELECT
    ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'production_db';
```

## Include Free Space (Fragmentation) Per Database

```sql
SELECT
    TABLE_SCHEMA AS database_name,
    ROUND(SUM(DATA_LENGTH) / 1024 / 1024, 2) AS data_mb,
    ROUND(SUM(INDEX_LENGTH) / 1024 / 1024, 2) AS index_mb,
    ROUND(SUM(DATA_FREE) / 1024 / 1024, 2) AS free_mb
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA NOT IN (
    'information_schema', 'performance_schema', 'mysql', 'sys'
)
GROUP BY TABLE_SCHEMA
ORDER BY data_mb DESC;
```

## Find Databases Above a Size Threshold

Alert when a database exceeds a certain size, useful in monitoring scripts:

```sql
SELECT
    TABLE_SCHEMA AS database_name,
    ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1073741824, 2) AS size_gb
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA NOT IN (
    'information_schema', 'performance_schema', 'mysql', 'sys'
)
GROUP BY TABLE_SCHEMA
HAVING size_gb > 10
ORDER BY size_gb DESC;
```

## Compare Data vs Index Ratio Across Databases

Databases where indexes are close to or larger than data may be over-indexed:

```sql
SELECT
    TABLE_SCHEMA AS database_name,
    ROUND(SUM(DATA_LENGTH) / 1024 / 1024, 1) AS data_mb,
    ROUND(SUM(INDEX_LENGTH) / 1024 / 1024, 1) AS index_mb,
    ROUND(SUM(INDEX_LENGTH) / NULLIF(SUM(DATA_LENGTH), 0), 2) AS index_to_data_ratio
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA NOT IN (
    'information_schema', 'performance_schema', 'mysql', 'sys'
)
GROUP BY TABLE_SCHEMA
ORDER BY index_to_data_ratio DESC;
```

## Automate Size Reporting with a Shell Script

```bash
#!/bin/bash
mysql -u root -p -e "
SELECT
    TABLE_SCHEMA AS db,
    ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1048576, 2) AS size_mb
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema','performance_schema','mysql','sys')
GROUP BY TABLE_SCHEMA
ORDER BY size_mb DESC;
"
```

## Summary

Aggregating `DATA_LENGTH` and `INDEX_LENGTH` from `INFORMATION_SCHEMA.TABLES` gives you a practical, server-wide view of database sizes without needing filesystem access. Combine this with scheduled reports or monitoring queries to stay ahead of storage growth and optimize before capacity problems arise.
