# How to Configure read_rnd_buffer_size in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Configuration, Buffer, Sorting, Performance

Description: Learn how MySQL read_rnd_buffer_size controls sorted row read performance after ORDER BY operations and how to tune it for better query throughput.

---

## What Is read_rnd_buffer_size

After MySQL sorts rows using a sort buffer (for `ORDER BY` queries), it needs to read the actual row data back from the storage engine in sorted order. When reading sorted rows that are scattered across disk, `read_rnd_buffer_size` controls how many row pointers are batched together before reading the actual data.

Batching row pointer reads allows the storage engine to sort them by position and read them in a more sequential order, reducing random I/O.

## Check the Current Value

```sql
SHOW VARIABLES LIKE 'read_rnd_buffer_size';
```

```text
+----------------------+--------+
| Variable_name        | Value  |
+----------------------+--------+
| read_rnd_buffer_size | 262144 |
+----------------------+--------+
```

Default is 256 KB.

## When read_rnd_buffer_size Is Used

It applies when MySQL uses the `filesort` algorithm:

```sql
EXPLAIN SELECT * FROM orders ORDER BY customer_id LIMIT 1000\G
```

```text
  Extra: Using filesort
```

When `Using filesort` appears in EXPLAIN, the sorted reads are governed by `read_rnd_buffer_size`.

## Monitor Sorted Reads

```sql
SHOW GLOBAL STATUS LIKE 'Handler_read_rnd%';
```

```text
+----------------------+----------+
| Variable_name        | Value    |
+----------------------+----------+
| Handler_read_rnd     | 4523400  |
| Handler_read_rnd_next| 12034521 |
+----------------------+----------+
```

`Handler_read_rnd` counts random position reads (sorted row fetches). High values suggest sorted queries are common and `read_rnd_buffer_size` may be relevant.

## Increase read_rnd_buffer_size

```sql
SET GLOBAL read_rnd_buffer_size = 1048576;  -- 1 MB
```

In `my.cnf`:

```ini
[mysqld]
read_rnd_buffer_size = 1M
```

## Set Per Session for Sort-Heavy Queries

```sql
SET SESSION read_rnd_buffer_size = 8388608;  -- 8 MB
SELECT *
FROM large_orders_table
ORDER BY created_at DESC, customer_id ASC
LIMIT 50000;
```

## Relationship with sort_buffer_size

These two variables work together in the sort pipeline:

1. `sort_buffer_size` - holds row keys and pointers during the sort phase
2. `read_rnd_buffer_size` - batches pointer reads to fetch actual rows in sorted order

For sort-heavy workloads, tuning both can improve performance:

```sql
-- Tune both for a reporting session
SET SESSION sort_buffer_size = 16777216;      -- 16 MB
SET SESSION read_rnd_buffer_size = 8388608;   -- 8 MB
```

## Avoid Sorts with Covering Indexes

The best way to improve sorted query performance is to add a covering index that returns rows in sorted order without a filesort:

```sql
-- Query causing filesort
SELECT id, name, created_at FROM users ORDER BY created_at DESC LIMIT 100;

-- Add a covering index
ALTER TABLE users ADD INDEX idx_covering (created_at DESC, id, name);

-- Verify sort is eliminated
EXPLAIN SELECT id, name, created_at FROM users ORDER BY created_at DESC LIMIT 100\G
-- Extra: Using index (no filesort)
```

## Summary

`read_rnd_buffer_size` controls the buffer used for reading sorted rows back from storage after `ORDER BY` operations. It is relevant when EXPLAIN shows `Using filesort`. Increase it globally or per session for sort-heavy workloads, keeping it balanced with `sort_buffer_size`. The best long-term fix is adding indexes to eliminate filesorts, but tuning `read_rnd_buffer_size` helps when sorts cannot be avoided.
