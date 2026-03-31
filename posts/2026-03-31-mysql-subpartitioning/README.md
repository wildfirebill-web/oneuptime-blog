# How to Use Subpartitioning in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Partition, Subpartition, Performance, InnoDB

Description: Learn how to use subpartitioning (composite partitioning) in MySQL to divide partitions into smaller subpartitions for finer-grained data management.

---

## What Is Subpartitioning?

Subpartitioning, also called composite partitioning, allows each partition of a table to be further divided into subpartitions. MySQL supports subpartitioning only in specific combinations:

- RANGE or LIST partitioning at the top level
- HASH or KEY subpartitioning at the second level

This gives you two-dimensional partitioning: data is first routed to a partition by one criterion (like a date range), then to a subpartition by another (like a hash of a user ID). The result is a finer-grained physical layout that can improve parallel query performance and maintenance operations.

## Basic Syntax

```sql
CREATE TABLE orders (
    order_id BIGINT NOT NULL,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    amount DECIMAL(10, 2),
    PRIMARY KEY (order_id, order_date, customer_id)
)
PARTITION BY RANGE (YEAR(order_date))
SUBPARTITION BY HASH (customer_id)
SUBPARTITIONS 4
(
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

This creates 4 partitions x 4 subpartitions = 16 physical partition files.

## RANGE + KEY Subpartitioning

```sql
CREATE TABLE events (
    event_id BIGINT NOT NULL,
    region_id INT NOT NULL,
    event_date DATE NOT NULL,
    payload TEXT,
    PRIMARY KEY (event_id, region_id, event_date)
)
PARTITION BY RANGE (YEAR(event_date))
SUBPARTITION BY KEY (region_id)
SUBPARTITIONS 8
(
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

## Naming Subpartitions Explicitly

You can name each subpartition individually for more control:

```sql
CREATE TABLE logs (
    log_id BIGINT NOT NULL,
    app_id INT NOT NULL,
    log_date DATE NOT NULL,
    message TEXT,
    PRIMARY KEY (log_id, app_id, log_date)
)
PARTITION BY RANGE (YEAR(log_date))
SUBPARTITION BY HASH (app_id)
(
    PARTITION p2024 VALUES LESS THAN (2025) (
        SUBPARTITION s0,
        SUBPARTITION s1,
        SUBPARTITION s2,
        SUBPARTITION s3
    ),
    PARTITION p2025 VALUES LESS THAN (2026) (
        SUBPARTITION s4,
        SUBPARTITION s5,
        SUBPARTITION s6,
        SUBPARTITION s7
    )
);
```

## Querying Subpartition Metadata

```sql
SELECT
    PARTITION_NAME,
    SUBPARTITION_NAME,
    PARTITION_METHOD,
    SUBPARTITION_METHOD,
    TABLE_ROWS
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'orders'
ORDER BY PARTITION_NAME, SUBPARTITION_NAME;
```

## Rules and Constraints

- All partitions must have the same number of subpartitions
- Subpartitions must use HASH or KEY
- The subpartitioning expression column must be in the primary key
- You cannot mix explicitly named and unnamed subpartitions within the same table

## When to Use Subpartitioning

- Tables that are very large and benefit from finer physical file granularity
- When you want to archive entire partitions (date ranges) while also spreading load across multiple files within each range
- Environments where tablespace-level operations benefit from smaller units

## Summary

Subpartitioning in MySQL provides a two-level partitioning hierarchy that combines the strengths of RANGE/LIST partitioning for range-based management with HASH/KEY partitioning for even distribution within each range. Use it when a single partition level produces partitions that are still too large for efficient management or parallel access.
