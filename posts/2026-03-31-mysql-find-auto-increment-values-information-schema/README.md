# How to Find All Auto-Increment Values Using INFORMATION_SCHEMA in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, INFORMATION_SCHEMA, Auto-Increment, Table, Database Administration

Description: Learn how to find the current auto-increment values of all tables in MySQL using INFORMATION_SCHEMA.TABLES to monitor ID consumption and detect near-overflow conditions.

---

## Why Monitor Auto-Increment Values?

Auto-increment columns have a maximum value determined by their data type. An `INT` column maxes out at 2,147,483,647 (signed) or 4,294,967,295 (unsigned). When the counter reaches that limit, inserts fail with a duplicate key error. Proactively monitoring how close each table's auto-increment counter is to its limit prevents unexpected outages.

`INFORMATION_SCHEMA.TABLES` exposes the `AUTO_INCREMENT` column, which shows the next value that will be assigned.

## Find All Tables with Auto-Increment Counters

```sql
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    AUTO_INCREMENT
FROM INFORMATION_SCHEMA.TABLES
WHERE AUTO_INCREMENT IS NOT NULL
  AND TABLE_SCHEMA NOT IN (
      'information_schema', 'performance_schema', 'mysql', 'sys'
  )
ORDER BY AUTO_INCREMENT DESC;
```

## Find the Highest Auto-Increment Values

```sql
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    AUTO_INCREMENT
FROM INFORMATION_SCHEMA.TABLES
WHERE AUTO_INCREMENT IS NOT NULL
  AND TABLE_SCHEMA NOT IN (
      'information_schema', 'performance_schema', 'mysql', 'sys'
  )
ORDER BY AUTO_INCREMENT DESC
LIMIT 10;
```

## Detect Tables Near INT Overflow

For signed `INT` columns (max 2,147,483,647):

```sql
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    AUTO_INCREMENT,
    ROUND(AUTO_INCREMENT / 2147483647 * 100, 2) AS pct_used
FROM INFORMATION_SCHEMA.TABLES
WHERE AUTO_INCREMENT IS NOT NULL
  AND TABLE_SCHEMA NOT IN (
      'information_schema', 'performance_schema', 'mysql', 'sys'
  )
  AND AUTO_INCREMENT > 1000000000
ORDER BY AUTO_INCREMENT DESC;
```

Tables above 50% usage should be monitored; those above 80% need urgent attention.

## Check the Data Type of Auto-Increment Columns

Combine `TABLES` and `COLUMNS` to see the column type alongside the counter:

```sql
SELECT
    t.TABLE_SCHEMA,
    t.TABLE_NAME,
    c.COLUMN_NAME,
    c.COLUMN_TYPE,
    t.AUTO_INCREMENT
FROM INFORMATION_SCHEMA.TABLES t
JOIN INFORMATION_SCHEMA.COLUMNS c
    ON t.TABLE_SCHEMA = c.TABLE_SCHEMA
    AND t.TABLE_NAME = c.TABLE_NAME
    AND c.EXTRA = 'auto_increment'
WHERE t.AUTO_INCREMENT IS NOT NULL
  AND t.TABLE_SCHEMA NOT IN (
      'information_schema', 'performance_schema', 'mysql', 'sys'
  )
ORDER BY t.AUTO_INCREMENT DESC;
```

## Calculate Remaining Capacity by Column Type

```sql
SELECT
    t.TABLE_SCHEMA,
    t.TABLE_NAME,
    c.COLUMN_TYPE,
    t.AUTO_INCREMENT,
    CASE
        WHEN c.COLUMN_TYPE = 'int' THEN 2147483647 - t.AUTO_INCREMENT
        WHEN c.COLUMN_TYPE = 'int unsigned' THEN 4294967295 - t.AUTO_INCREMENT
        WHEN c.COLUMN_TYPE = 'bigint' THEN 9223372036854775807 - t.AUTO_INCREMENT
        WHEN c.COLUMN_TYPE = 'bigint unsigned' THEN 18446744073709551615 - t.AUTO_INCREMENT
        ELSE NULL
    END AS remaining_ids
FROM INFORMATION_SCHEMA.TABLES t
JOIN INFORMATION_SCHEMA.COLUMNS c
    ON t.TABLE_SCHEMA = c.TABLE_SCHEMA
    AND t.TABLE_NAME = c.TABLE_NAME
    AND c.EXTRA = 'auto_increment'
WHERE t.AUTO_INCREMENT IS NOT NULL
  AND t.TABLE_SCHEMA NOT IN (
      'information_schema', 'performance_schema', 'mysql', 'sys'
  )
ORDER BY remaining_ids ASC;
```

## Fix a Near-Overflow: Migrate to BIGINT

If a table is approaching INT limits, alter the column:

```sql
ALTER TABLE orders
MODIFY COLUMN id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT;
```

## Summary

`INFORMATION_SCHEMA.TABLES.AUTO_INCREMENT` gives you visibility into ID consumption across your entire database server. Join it with `INFORMATION_SCHEMA.COLUMNS` to factor in the column data type and calculate exactly how much runway remains before an overflow. Set up monitoring alerts when any table exceeds 70-80% of its maximum ID capacity.
