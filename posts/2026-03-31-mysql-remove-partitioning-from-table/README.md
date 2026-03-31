# How to Remove Partitioning from a Table in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Partition, ALTER TABLE, Migration, InnoDB

Description: Learn how to remove partitioning from a MySQL table using ALTER TABLE REMOVE PARTITIONING, and understand the implications for data, indexes, and primary keys.

---

## Why Remove Partitioning?

Sometimes partitioning turns out to be the wrong choice for a table. If query patterns rarely align with partition boundaries, partition pruning provides no benefit, and the partitioning overhead adds complexity without reward. In these cases, removing partitioning returns the table to a standard, non-partitioned structure.

Other reasons to remove partitioning include:
- Migrating to a different partitioning strategy (remove and re-add)
- Simplifying schema for applications that cannot handle partitioned tables well
- Consolidating data after all historical partitions have been archived

## The REMOVE PARTITIONING Command

MySQL provides a clean, single-statement way to remove partitioning:

```sql
ALTER TABLE orders REMOVE PARTITIONING;
```

This operation:
- Merges all partitions into a single unified table
- Preserves all data - no rows are deleted
- Keeps all existing indexes and constraints
- Does NOT change the primary key

## Full Example

Start with a partitioned table:

```sql
CREATE TABLE sales (
    sale_id BIGINT NOT NULL,
    sale_date DATE NOT NULL,
    amount DECIMAL(10, 2),
    PRIMARY KEY (sale_id, sale_date)
)
PARTITION BY RANGE (YEAR(sale_date))
(
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);

-- Check current partitions
SELECT PARTITION_NAME, TABLE_ROWS
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'sales';

-- Remove partitioning
ALTER TABLE sales REMOVE PARTITIONING;

-- Verify: only one NULL partition entry now
SELECT PARTITION_NAME, TABLE_ROWS
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'sales';
```

After removal, `PARTITION_NAME` will be `NULL`, indicating a non-partitioned table.

## Verify the Table Structure

```sql
SHOW CREATE TABLE sales\G
```

The output should no longer contain any `PARTITION BY` clause.

## Row Count Verification

Always verify that data was preserved:

```sql
-- Before removal
SELECT COUNT(*) FROM sales;

-- After removal
SELECT COUNT(*) FROM sales;
```

## Primary Key After Removal

`REMOVE PARTITIONING` does not touch the primary key. If you added `sale_date` to the primary key specifically to satisfy partitioning requirements, you can now optionally simplify it:

```sql
ALTER TABLE sales
DROP PRIMARY KEY,
ADD PRIMARY KEY (sale_id);
```

Be careful: removing a column from the primary key may have implications for replication and application logic.

## Performance Impact During Removal

`REMOVE PARTITIONING` rebuilds the entire table by merging all partitions into one. For large tables, this is a time-consuming and I/O-intensive operation. Estimate the time based on total table size:

```sql
SELECT
    ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1073741824, 2) AS total_gb
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'sales';
```

For very large tables, consider using `pt-online-schema-change` to avoid extended locking.

## Removing Partitioning as a Migration Step

A common pattern when changing partitioning strategy:

```sql
-- Step 1: Remove old partitioning
ALTER TABLE events REMOVE PARTITIONING;

-- Step 2: Apply new partitioning scheme
ALTER TABLE events
PARTITION BY RANGE (TO_DAYS(event_date))
(
    PARTITION p_2025q1 VALUES LESS THAN (TO_DAYS('2025-04-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

## Summary

`ALTER TABLE ... REMOVE PARTITIONING` safely merges a partitioned MySQL table back into a standard single-segment table while preserving all data and indexes. It is the cleanest way to undo partitioning or reset the table before applying a new partitioning strategy. Plan for the full table rebuild time and consider online schema change tools for large production tables.
