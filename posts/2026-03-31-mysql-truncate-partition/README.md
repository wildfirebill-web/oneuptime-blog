# How to Truncate a Partition in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Partition, ALTER TABLE, Data Management, InnoDB

Description: Learn how to truncate a MySQL partition using ALTER TABLE TRUNCATE PARTITION to quickly delete all rows while keeping the partition structure intact.

---

## What Is TRUNCATE PARTITION?

`ALTER TABLE ... TRUNCATE PARTITION` deletes all rows from one or more partitions without removing the partition definitions themselves. The partition structure remains in place and continues to accept new inserts. This makes it ideal for resetting data in a specific range while keeping the partitioned table schema intact.

This operation is much faster than `DELETE` because MySQL drops and recreates the underlying data file rather than scanning rows individually.

## TRUNCATE PARTITION vs DROP PARTITION

| Operation | Removes Data | Removes Partition Definition |
|---|---|---|
| TRUNCATE PARTITION | Yes | No |
| DROP PARTITION | Yes | Yes |

Use `TRUNCATE PARTITION` when you want to keep the partition structure for future data. Use `DROP PARTITION` when the partition is no longer needed at all.

## Basic Syntax

```sql
ALTER TABLE orders
TRUNCATE PARTITION p2022;
```

## Truncate Multiple Partitions

```sql
ALTER TABLE orders
TRUNCATE PARTITION p2021, p2022;
```

## Truncate All Partitions

```sql
ALTER TABLE orders
TRUNCATE PARTITION ALL;
```

This is equivalent to `TRUNCATE TABLE` but operates partition by partition.

## Practical Example: Monthly Log Rotation

Create a table with monthly partitions for log rotation:

```sql
CREATE TABLE app_logs (
    log_id BIGINT NOT NULL AUTO_INCREMENT,
    log_month INT NOT NULL,
    severity VARCHAR(10),
    message TEXT,
    created_at DATETIME,
    PRIMARY KEY (log_id, log_month)
)
PARTITION BY LIST (log_month)
(
    PARTITION p01 VALUES IN (1),
    PARTITION p02 VALUES IN (2),
    PARTITION p03 VALUES IN (3),
    PARTITION p04 VALUES IN (4),
    PARTITION p05 VALUES IN (5),
    PARTITION p06 VALUES IN (6),
    PARTITION p07 VALUES IN (7),
    PARTITION p08 VALUES IN (8),
    PARTITION p09 VALUES IN (9),
    PARTITION p10 VALUES IN (10),
    PARTITION p11 VALUES IN (11),
    PARTITION p12 VALUES IN (12)
);
```

At the start of each month, truncate the partition for that month to clear old data:

```sql
-- At the start of March, clear the March partition from last year
ALTER TABLE app_logs
TRUNCATE PARTITION p03;
```

New March data then fills the clean partition.

## Check Row Count Before Truncating

```sql
SELECT
    PARTITION_NAME,
    TABLE_ROWS
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'app_logs'
ORDER BY PARTITION_NAME;
```

## Verify After Truncation

```sql
-- Confirm the partition is empty
SELECT COUNT(*) FROM app_logs
PARTITION (p03);
```

## Automating Partition Truncation

```sql
DELIMITER //
CREATE EVENT rotate_log_partition
ON SCHEDULE EVERY 1 MONTH
STARTS '2025-03-01 00:00:01'
DO BEGIN
    SET @month_num = MONTH(CURDATE());
    SET @sql = CONCAT(
        'ALTER TABLE app_logs TRUNCATE PARTITION p',
        LPAD(@month_num, 2, '0')
    );
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
END//
DELIMITER ;
```

## Summary

`TRUNCATE PARTITION` provides a fast, lightweight way to clear partition data while preserving the partition layout. It is the foundation of partition-based log rotation strategies and is orders of magnitude faster than `DELETE FROM table WHERE date_column BETWEEN ...`. Use it wherever you need recurring data purges without restructuring the table.
