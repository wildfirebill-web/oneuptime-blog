# How to Use ALTER TABLE ... DROP PARTITION in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Partition, DDL, Table, Archiving

Description: Learn how to use ALTER TABLE ... DROP PARTITION in MySQL to efficiently delete old partitions and reclaim storage without full table scans.

---

## Overview

`ALTER TABLE ... DROP PARTITION` removes one or more partitions from a partitioned table, along with all data stored in those partitions. This operation is much faster than `DELETE` for removing large volumes of old data because MySQL does not scan rows individually - it simply drops the partition's underlying storage files.

## Basic Syntax

```sql
ALTER TABLE table_name DROP PARTITION partition_name [, partition_name ...];
```

## Why Use DROP PARTITION Instead of DELETE

For partitioned tables, dropping a partition is an instantaneous DDL operation compared to a row-by-row `DELETE`:

- `DELETE FROM sales WHERE sale_year = 2020` - scans all rows, generates undo log entries, slow
- `ALTER TABLE sales DROP PARTITION p2020` - removes the entire partition file, near-instant

## Simple Example

Given a range-partitioned sales table:

```sql
CREATE TABLE sales (
  id        INT NOT NULL AUTO_INCREMENT,
  amount    DECIMAL(10,2) NOT NULL,
  sale_year INT NOT NULL,
  PRIMARY KEY (id, sale_year)
)
PARTITION BY RANGE (sale_year) (
  PARTITION p2020 VALUES LESS THAN (2021),
  PARTITION p2021 VALUES LESS THAN (2022),
  PARTITION p2022 VALUES LESS THAN (2023),
  PARTITION p2023 VALUES LESS THAN (2024)
);
```

Drop the oldest partition to purge 2020 data:

```sql
ALTER TABLE sales DROP PARTITION p2020;
```

All rows from the 2020 partition are permanently deleted.

## Dropping Multiple Partitions at Once

```sql
ALTER TABLE sales DROP PARTITION p2020, p2021;
```

This removes both partitions in a single operation.

## Checking Partitions Before Dropping

Always verify which partition holds which data before dropping:

```sql
SELECT PARTITION_NAME, PARTITION_DESCRIPTION, TABLE_ROWS
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'sales'
ORDER BY PARTITION_ORDINAL_POSITION;
```

## Archiving Data Before Dropping

If you need to archive data before deletion, create an archive table, copy the data, then drop the partition:

```sql
-- Create archive table with same structure
CREATE TABLE sales_archive_2020 LIKE sales;
ALTER TABLE sales_archive_2020 REMOVE PARTITIONING;

-- Copy data
INSERT INTO sales_archive_2020
SELECT * FROM sales PARTITION (p2020);

-- Drop the partition
ALTER TABLE sales DROP PARTITION p2020;
```

## LIST Partitioned Tables

`DROP PARTITION` also works on `LIST` partitioned tables:

```sql
ALTER TABLE orders DROP PARTITION p_apac;
```

All rows whose region value mapped to `p_apac` are deleted.

## Important Limitations

- `DROP PARTITION` permanently deletes all data in the partition - there is no rollback
- It cannot be used on `HASH` or `KEY` partitioned tables (use `COALESCE PARTITION` instead)
- The table must have at least one partition remaining after the drop

## Rolling Partition Strategy

A common pattern is to combine `ADD PARTITION` and `DROP PARTITION` in a rolling window:

```sql
-- Monthly maintenance job: add next month, drop oldest
ALTER TABLE events
  ADD PARTITION (
    PARTITION p_2026_04 VALUES LESS THAN ('2026-05-01')
  );

ALTER TABLE events DROP PARTITION p_2025_04;
```

## Summary

`ALTER TABLE ... DROP PARTITION` is the most efficient way to purge large volumes of partitioned data. It removes entire partition files rather than deleting rows individually, making it orders of magnitude faster than `DELETE` for historical data cleanup. Use it as part of a rolling partition maintenance strategy to keep table sizes manageable.
