# How MySQL Partitioning Works Internally

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Partitioning, Performance, InnoDB, Database Design

Description: Learn how MySQL table partitioning divides large tables into logical segments, how the optimizer uses partition pruning, and which partition types to use.

---

## What Is MySQL Table Partitioning

Partitioning divides a single logical table into multiple physical storage segments (partitions) based on a partitioning key. Each partition is stored as a separate InnoDB tablespace file, but queries access the table normally without knowing about the partitions.

Partitioning benefits:
- **Partition pruning** - the optimizer skips partitions that cannot contain relevant data.
- **Faster data deletion** - drop entire partitions instead of deleting rows.
- **Manageability** - archive old data by dropping old partitions.

## Partition Types

### RANGE Partitioning

Divides rows into partitions based on column value ranges:

```sql
CREATE TABLE orders (
  id INT AUTO_INCREMENT,
  customer_id INT,
  order_date DATE,
  amount DECIMAL(10,2),
  PRIMARY KEY (id, order_date)  -- Partition key must be in PK
)
PARTITION BY RANGE (YEAR(order_date)) (
  PARTITION p2023 VALUES LESS THAN (2024),
  PARTITION p2024 VALUES LESS THAN (2025),
  PARTITION p2025 VALUES LESS THAN (2026),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

### LIST Partitioning

Divides rows based on a list of discrete values:

```sql
CREATE TABLE employees (
  id INT AUTO_INCREMENT,
  name VARCHAR(100),
  region_id INT,
  PRIMARY KEY (id, region_id)
)
PARTITION BY LIST (region_id) (
  PARTITION p_north  VALUES IN (1, 2, 3),
  PARTITION p_south  VALUES IN (4, 5, 6),
  PARTITION p_east   VALUES IN (7, 8, 9),
  PARTITION p_west   VALUES IN (10, 11, 12)
);
```

### HASH Partitioning

Distributes rows evenly across N partitions using a hash:

```sql
CREATE TABLE sessions (
  id INT AUTO_INCREMENT,
  user_id INT,
  created_at TIMESTAMP,
  PRIMARY KEY (id, user_id)
)
PARTITION BY HASH (user_id)
PARTITIONS 8;
```

### KEY Partitioning

Similar to HASH but MySQL chooses the hash function:

```sql
PARTITION BY KEY (user_id)
PARTITIONS 8;
```

## How Partition Pruning Works

Partition pruning is the optimizer's ability to skip irrelevant partitions. For it to work, the WHERE clause must reference the partitioning column:

```sql
-- This query uses partition pruning (only reads p2025)
EXPLAIN SELECT * FROM orders WHERE order_date BETWEEN '2025-01-01' AND '2025-12-31';
```

```text
+------+-------+----------+------------------+
| type | table | rows     | partitions       |
+------+-------+----------+------------------+
| ALL  | orders| 50000    | p2025            |  -- Only p2025 scanned
+------+-------+----------+------------------+
```

```sql
-- This query does NOT use pruning (full table scan)
SELECT * FROM orders WHERE amount > 100;
-- Partition key (order_date) is not in WHERE - all partitions scanned
```

## Adding and Dropping Partitions

Adding a new partition for the next year:

```sql
ALTER TABLE orders
  REORGANIZE PARTITION p_future INTO (
    PARTITION p2026 VALUES LESS THAN (2027),
    PARTITION p_future VALUES LESS THAN MAXVALUE
  );
```

Dropping an old partition (instant - no row-by-row deletion):

```sql
ALTER TABLE orders DROP PARTITION p2023;
```

Dropping a partition is orders of magnitude faster than `DELETE FROM orders WHERE YEAR(order_date) = 2023`.

## Viewing Partition Information

```sql
SELECT
  PARTITION_NAME,
  TABLE_ROWS,
  DATA_LENGTH,
  INDEX_LENGTH,
  PARTITION_EXPRESSION
FROM information_schema.PARTITIONS
WHERE TABLE_SCHEMA = 'myapp'
  AND TABLE_NAME = 'orders';
```

## Partition Limitations

- The partitioning key must be part of every unique index (including PRIMARY KEY).
- Foreign keys are not supported on partitioned tables.
- Full-text indexes are not supported.
- MyISAM and InnoDB are the only engines with full partitioning support in older MySQL versions.
- Maximum 8192 partitions per table.

## Subpartitioning

You can further divide partitions into subpartitions:

```sql
CREATE TABLE orders_sub (
  id INT,
  order_date DATE,
  region_id INT,
  PRIMARY KEY (id, order_date, region_id)
)
PARTITION BY RANGE (YEAR(order_date))
SUBPARTITION BY HASH (region_id) SUBPARTITIONS 4 (
  PARTITION p2024 VALUES LESS THAN (2025),
  PARTITION p2025 VALUES LESS THAN (2026)
);
```

## When to Use Partitioning

Partitioning is most beneficial when:
- Tables exceed tens of millions of rows and grow indefinitely (time-series data).
- You frequently query data by the partitioning key.
- You need to purge old data quickly (partition drop).

Partitioning is NOT a substitute for proper indexing. Always add appropriate indexes within partitions for non-partition key queries.

## Summary

MySQL partitioning splits large tables into physical segments for faster queries via partition pruning, instant data archival via partition drops, and improved manageability. RANGE partitioning is the most common pattern for time-series data. The optimizer automatically prunes irrelevant partitions when the WHERE clause includes the partition key. Always ensure the partition key is part of every unique index, and use partitioning primarily for tables with clear range or list distribution patterns.
