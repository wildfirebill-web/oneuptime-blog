# How to Add a Partition to an Existing Table in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Partition, ALTER TABLE, Performance, InnoDB

Description: Learn how to add new partitions to an existing partitioned MySQL table using ALTER TABLE, with examples for RANGE, LIST, and HASH partitioning.

---

## Why Add Partitions to an Existing Table?

Partitioned tables need new partitions over time. For RANGE-partitioned tables (often date-based), you regularly add new partitions to hold future data. For HASH or KEY tables, you may need to add partitions as data volume grows. Adding partitions without rebuilding the entire table is done with `ALTER TABLE ... ADD PARTITION`.

## Adding a Partition to a RANGE Table

The most common use case is adding a new time-based partition at the end of a RANGE-partitioned table.

First, create a table with initial partitions:

```sql
CREATE TABLE sales (
    sale_id INT NOT NULL,
    sale_date DATE NOT NULL,
    amount DECIMAL(10, 2),
    PRIMARY KEY (sale_id, sale_date)
)
PARTITION BY RANGE (YEAR(sale_date))
(
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

To add a 2025 partition, you must first reorganize the `p_future` catch-all partition:

```sql
ALTER TABLE sales
REORGANIZE PARTITION p_future INTO (
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

If the table has no catch-all `MAXVALUE` partition, you can simply append:

```sql
CREATE TABLE events (
    event_id INT NOT NULL,
    event_year INT NOT NULL,
    PRIMARY KEY (event_id, event_year)
)
PARTITION BY RANGE (event_year)
(
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);

-- Add a new partition directly
ALTER TABLE events
ADD PARTITION (
    PARTITION p2025 VALUES LESS THAN (2026)
);
```

## Adding a Partition to a LIST Table

```sql
CREATE TABLE regions (
    id INT NOT NULL,
    region_code INT NOT NULL,
    name VARCHAR(100),
    PRIMARY KEY (id, region_code)
)
PARTITION BY LIST (region_code)
(
    PARTITION p_americas VALUES IN (1, 2, 3),
    PARTITION p_europe VALUES IN (10, 11, 12)
);

-- Add a new region group
ALTER TABLE regions
ADD PARTITION (
    PARTITION p_asia VALUES IN (20, 21, 22)
);
```

## Adding Partitions to a HASH or KEY Table

For HASH and KEY partitions, use `ADD PARTITION PARTITIONS n` to increase the count:

```sql
CREATE TABLE sessions (
    session_id BIGINT NOT NULL,
    user_id INT NOT NULL,
    PRIMARY KEY (session_id)
)
PARTITION BY HASH (session_id)
PARTITIONS 4;

-- Increase from 4 to 8 partitions
ALTER TABLE sessions
ADD PARTITION PARTITIONS 4;
```

Note: this operation redistributes data across the new partition count and can be slow on large tables.

## Verify the New Partition Exists

```sql
SELECT
    PARTITION_NAME,
    PARTITION_DESCRIPTION,
    TABLE_ROWS
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'sales'
ORDER BY PARTITION_ORDINAL_POSITION;
```

## Automate Partition Addition for Time-Series Tables

A common pattern is a stored event that runs monthly:

```sql
DELIMITER //
CREATE EVENT add_monthly_partition
ON SCHEDULE EVERY 1 MONTH
STARTS '2025-12-01 00:00:00'
DO BEGIN
    SET @next_year = YEAR(CURDATE() + INTERVAL 1 MONTH);
    SET @next_month = MONTH(CURDATE() + INTERVAL 1 MONTH);
    SET @sql = CONCAT(
        'ALTER TABLE logs ADD PARTITION (',
        'PARTITION p', @next_year, '_', LPAD(@next_month, 2, '0'),
        ' VALUES LESS THAN (', @next_year * 100 + @next_month, '))'
    );
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
END//
DELIMITER ;
```

## Summary

Adding partitions to existing MySQL tables is straightforward for LIST and HASH/KEY partitioning. For RANGE tables with a `MAXVALUE` catch-all, use `REORGANIZE PARTITION` to insert a new partition before the catch-all. Automating partition addition via MySQL Events keeps time-series tables properly partitioned without manual intervention.
