# How to Use LINEAR HASH Partitioning in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Partition, Linear Hash, Performance, InnoDB

Description: Learn how to use LINEAR HASH partitioning in MySQL, how it differs from regular HASH partitioning, and when to choose it for large tables.

---

## What Is LINEAR HASH Partitioning?

LINEAR HASH partitioning is a variant of standard HASH partitioning that uses a linear powers-of-two algorithm to determine the partition for each row. The key difference is that LINEAR HASH uses a faster algorithm that makes it easier to add or remove partitions - operations that are faster than with regular HASH because fewer rows need to be moved between partitions.

The trade-off is that LINEAR HASH produces less uniform data distribution compared to regular HASH.

## Syntax

```sql
CREATE TABLE orders (
    order_id INT NOT NULL,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    amount DECIMAL(10, 2),
    PRIMARY KEY (order_id)
)
PARTITION BY LINEAR HASH (order_id)
PARTITIONS 8;
```

## LINEAR HASH vs Regular HASH

With regular HASH, MySQL computes `MOD(expr, num_partitions)` to determine the partition. With LINEAR HASH, MySQL uses a powers-of-two algorithm:

```text
Regular HASH: partition = MOD(order_id, 8)
LINEAR HASH:  partition = determined by next power of 2 >= num_partitions
```

The practical difference appears when adding partitions. Regular HASH requires redistributing nearly all data; LINEAR HASH only moves a fraction of rows.

## Creating a LINEAR HASH Table with Multiple Columns

```sql
CREATE TABLE user_events (
    event_id BIGINT NOT NULL AUTO_INCREMENT,
    user_id INT NOT NULL,
    event_type VARCHAR(50),
    created_at DATETIME NOT NULL,
    PRIMARY KEY (event_id, user_id)
)
PARTITION BY LINEAR HASH (user_id)
PARTITIONS 16;
```

## Adding Partitions to a LINEAR HASH Table

Adding partitions to LINEAR HASH is faster because only a subset of data needs to be redistributed:

```sql
ALTER TABLE user_events
ADD PARTITION PARTITIONS 8;
-- Grows from 16 to 24 partitions
```

## Verifying Partition Distribution

After inserting data, check how evenly rows are distributed:

```sql
SELECT
    PARTITION_NAME,
    TABLE_ROWS
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'user_events'
ORDER BY PARTITION_NAME;
```

Note that LINEAR HASH typically produces less uniform distribution than regular HASH - some partitions will have noticeably more rows than others.

## When to Use LINEAR HASH

Use LINEAR HASH when:
- You anticipate needing to add partitions frequently as data grows
- Faster partition management operations are more important than perfectly even distribution
- You have very large tables where redistribution time is a concern

Use regular HASH when:
- Even data distribution is the top priority
- The number of partitions is unlikely to change

## Dropping Linear Hash Partitions

```sql
ALTER TABLE user_events
COALESCE PARTITION 4;
-- Reduces partition count by 4
```

Note that you cannot use `DROP PARTITION` with HASH or LINEAR HASH - use `COALESCE PARTITION` instead.

## Summary

LINEAR HASH partitioning in MySQL provides a faster algorithm for partition management at the cost of potentially less uniform data distribution. It is most valuable for large, fast-growing tables where you need to add partitions over time without long maintenance windows. For static partition counts, regular HASH typically provides better balance.
