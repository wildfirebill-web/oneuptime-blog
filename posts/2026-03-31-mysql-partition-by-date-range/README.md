# How to Partition by Date Range in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Partition, Range, Date, Performance

Description: Learn how to partition MySQL tables by date range using RANGE partitioning on date columns to improve query performance and simplify data archiving.

---

## Why Partition by Date?

Date-range partitioning is one of the most common and effective partitioning strategies. Most time-series tables (logs, events, transactions, metrics) are queried and archived by date. Partitioning by date means queries that filter on a date range automatically skip irrelevant partitions - a concept called partition pruning. Archiving old data is as fast as dropping a partition.

## Partitioning by YEAR

The simplest date partition uses `YEAR()`:

```sql
CREATE TABLE transactions (
    txn_id BIGINT NOT NULL,
    txn_date DATE NOT NULL,
    customer_id INT NOT NULL,
    amount DECIMAL(12, 2),
    PRIMARY KEY (txn_id, txn_date)
)
PARTITION BY RANGE (YEAR(txn_date))
(
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

## Partitioning by Month Using UNIX_TIMESTAMP

For monthly granularity on a `DATETIME` column:

```sql
CREATE TABLE web_logs (
    log_id BIGINT NOT NULL,
    log_time DATETIME NOT NULL,
    url VARCHAR(500),
    status_code SMALLINT,
    PRIMARY KEY (log_id, log_time)
)
PARTITION BY RANGE (UNIX_TIMESTAMP(log_time))
(
    PARTITION p202501 VALUES LESS THAN (UNIX_TIMESTAMP('2025-02-01')),
    PARTITION p202502 VALUES LESS THAN (UNIX_TIMESTAMP('2025-03-01')),
    PARTITION p202503 VALUES LESS THAN (UNIX_TIMESTAMP('2025-04-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

## Partitioning by DATE Column Using TO_DAYS

`TO_DAYS()` converts a DATE to an integer representing days since year 0, which works well for RANGE partitioning:

```sql
CREATE TABLE orders (
    order_id BIGINT NOT NULL,
    order_date DATE NOT NULL,
    amount DECIMAL(10, 2),
    PRIMARY KEY (order_id, order_date)
)
PARTITION BY RANGE (TO_DAYS(order_date))
(
    PARTITION p_2024q1 VALUES LESS THAN (TO_DAYS('2024-04-01')),
    PARTITION p_2024q2 VALUES LESS THAN (TO_DAYS('2024-07-01')),
    PARTITION p_2024q3 VALUES LESS THAN (TO_DAYS('2024-10-01')),
    PARTITION p_2024q4 VALUES LESS THAN (TO_DAYS('2025-01-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

## Query Performance with Partition Pruning

When you query with a date filter, MySQL automatically scans only the relevant partitions:

```sql
-- Only accesses the 2024 Q1 partition
SELECT * FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-03-31';
```

Verify which partitions are accessed with `EXPLAIN`:

```sql
EXPLAIN SELECT * FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-03-31'\G
```

Look for the `partitions` column in the output.

## Adding Future Partitions

Before the catch-all partition fills with new year data:

```sql
ALTER TABLE orders
REORGANIZE PARTITION p_future INTO (
    PARTITION p_2025q1 VALUES LESS THAN (TO_DAYS('2025-04-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

## Dropping Old Partitions

Archive data older than two years by simply dropping the partition:

```sql
ALTER TABLE orders
DROP PARTITION p_2022q1, p_2022q2, p_2022q3, p_2022q4;
```

This removes all matching rows without scanning, in milliseconds.

## Summary

Date-range partitioning is the standard approach for time-series data in MySQL. Using `YEAR()`, `TO_DAYS()`, or `UNIX_TIMESTAMP()` as the partition expression, you can route rows into the right partition automatically. The result is faster queries through pruning, trivially fast historical data archiving, and reduced lock contention on recent-data inserts.
