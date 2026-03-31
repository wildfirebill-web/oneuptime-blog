# How to Reorganize Partitions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Partition, ALTER TABLE, InnoDB, Database Administration

Description: Learn how to reorganize MySQL partitions using ALTER TABLE REORGANIZE PARTITION to split, merge, or restructure partition boundaries without data loss.

---

## What Is REORGANIZE PARTITION?

`ALTER TABLE ... REORGANIZE PARTITION` lets you restructure existing partitions by splitting them, merging them, or changing their boundaries - all without losing any data. MySQL reads all data from the affected partitions and redistributes it into the new partition definitions.

This is the safe way to adjust partition definitions when you cannot simply add or drop partitions.

## Common Use Cases

1. Splitting a `MAXVALUE` catch-all partition to insert a new explicit partition
2. Merging several small partitions into one
3. Splitting a large partition into smaller ones
4. Changing boundary values for RANGE or LIST partitions

## Split a MAXVALUE Partition (Most Common)

When you have a RANGE table with a catch-all partition:

```sql
CREATE TABLE logs (
    log_id BIGINT NOT NULL,
    log_date DATE NOT NULL,
    message TEXT,
    PRIMARY KEY (log_id, log_date)
)
PARTITION BY RANGE (YEAR(log_date))
(
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

To add a 2025 partition without losing the catch-all:

```sql
ALTER TABLE logs
REORGANIZE PARTITION p_future INTO (
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

## Merge Multiple Partitions into One

Consolidate several old partitions into a single archive partition:

```sql
ALTER TABLE logs
REORGANIZE PARTITION p2021, p2022, p2023 INTO (
    PARTITION p_archive VALUES LESS THAN (2024)
);
```

All data from the three partitions is moved into the new single partition.

## Split One Partition into Multiple

If a single partition has grown too large, split it:

```sql
-- Original: one partition covering a full year
-- Split into quarters
ALTER TABLE sales
REORGANIZE PARTITION p2024 INTO (
    PARTITION p2024_q1 VALUES LESS THAN (20240401),
    PARTITION p2024_q2 VALUES LESS THAN (20240701),
    PARTITION p2024_q3 VALUES LESS THAN (20241001),
    PARTITION p2024_q4 VALUES LESS THAN (20250101)
);
```

## Reorganize LIST Partitions

```sql
-- Original LIST partitioning
CREATE TABLE stores (
    store_id INT NOT NULL,
    region_code INT NOT NULL,
    PRIMARY KEY (store_id, region_code)
)
PARTITION BY LIST (region_code)
(
    PARTITION p_north VALUES IN (1, 2, 3, 4),
    PARTITION p_south VALUES IN (5, 6, 7, 8)
);

-- Reorganize: split p_north into two
ALTER TABLE stores
REORGANIZE PARTITION p_north INTO (
    PARTITION p_northeast VALUES IN (1, 2),
    PARTITION p_northwest VALUES IN (3, 4)
);
```

## Verify the Results

```sql
SELECT
    PARTITION_NAME,
    PARTITION_DESCRIPTION,
    TABLE_ROWS
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'logs'
ORDER BY PARTITION_ORDINAL_POSITION;
```

## Important Constraints

- The new partition definitions must cover exactly the same data range as the original partitions being reorganized
- No data is lost - MySQL reads all rows and redistributes them
- This operation can be slow on large tables because it copies data
- HASH and KEY partitions cannot use REORGANIZE PARTITION; use COALESCE and ADD instead

## Performance Consideration

`REORGANIZE PARTITION` rebuilds affected partitions. For large partitions, this can take significant time and generate substantial I/O. Consider running during low-traffic windows or using `pt-online-schema-change` for critical tables.

## Summary

`REORGANIZE PARTITION` is the flexible tool for restructuring partition boundaries in RANGE and LIST partitioned MySQL tables. Use it to insert new partitions into existing schemes, consolidate old partitions, or split large partitions as data distribution changes over time.
