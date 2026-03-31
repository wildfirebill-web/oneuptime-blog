# How to Use RANGE Partitioning in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Partitioning, Range Partitioning, Performance, Database Design

Description: Learn how to implement RANGE partitioning in MySQL to split large tables into logical segments by value ranges, with examples for dates and numeric columns.

---

## What Is RANGE Partitioning?

RANGE partitioning divides a table into partitions where each partition holds rows whose partitioning column value falls within a defined range. Ranges must be contiguous and non-overlapping.

This is most useful for tables containing date or time columns, where you want to separate data by year, month, or quarter.

## Basic Syntax

```sql
CREATE TABLE table_name (
  column_definitions
)
PARTITION BY RANGE (column_expr) (
  PARTITION p0 VALUES LESS THAN (value1),
  PARTITION p1 VALUES LESS THAN (value2),
  PARTITION pN VALUES LESS THAN MAXVALUE
);
```

## Example: Partitioning Sales by Year

```sql
CREATE TABLE sales (
  id INT NOT NULL AUTO_INCREMENT,
  sale_date DATE NOT NULL,
  customer_id INT NOT NULL,
  amount DECIMAL(10, 2) NOT NULL,
  PRIMARY KEY (id, sale_date)
)
PARTITION BY RANGE (YEAR(sale_date)) (
  PARTITION p2021 VALUES LESS THAN (2022),
  PARTITION p2022 VALUES LESS THAN (2023),
  PARTITION p2023 VALUES LESS THAN (2024),
  PARTITION p2024 VALUES LESS THAN (2025),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

## Example: Partitioning by Numeric Range

```sql
CREATE TABLE employees (
  id INT NOT NULL,
  name VARCHAR(100) NOT NULL,
  department_id INT NOT NULL,
  salary DECIMAL(10, 2)
)
PARTITION BY RANGE (department_id) (
  PARTITION p_dept_1_10   VALUES LESS THAN (11),
  PARTITION p_dept_11_20  VALUES LESS THAN (21),
  PARTITION p_dept_21_30  VALUES LESS THAN (31),
  PARTITION p_rest        VALUES LESS THAN MAXVALUE
);
```

## Using RANGE COLUMNS

`RANGE COLUMNS` allows partitioning on multiple columns or non-integer columns (strings, dates):

```sql
CREATE TABLE orders (
  order_id INT NOT NULL,
  order_date DATE NOT NULL,
  region VARCHAR(20) NOT NULL,
  total DECIMAL(10,2)
)
PARTITION BY RANGE COLUMNS (order_date) (
  PARTITION p_q1_2024 VALUES LESS THAN ('2024-04-01'),
  PARTITION p_q2_2024 VALUES LESS THAN ('2024-07-01'),
  PARTITION p_q3_2024 VALUES LESS THAN ('2024-10-01'),
  PARTITION p_q4_2024 VALUES LESS THAN ('2025-01-01'),
  PARTITION p_future  VALUES LESS THAN (MAXVALUE)
);
```

## Inserting Data

```sql
INSERT INTO sales (sale_date, customer_id, amount) VALUES
('2022-03-15', 101, 250.00),
('2023-08-20', 102, 400.00),
('2024-01-05', 103, 150.00);
```

Each row is automatically routed to the correct partition based on `sale_date`.

## Verifying Partition Placement

```sql
SELECT
  partition_name,
  table_rows
FROM information_schema.partitions
WHERE table_name = 'sales'
  AND table_schema = 'mydb';
```

## Querying a Specific Partition

```sql
SELECT * FROM sales PARTITION (p2022);
```

## Partition Pruning in Action

MySQL automatically prunes irrelevant partitions when a WHERE clause matches the partition key:

```sql
EXPLAIN SELECT * FROM sales WHERE YEAR(sale_date) = 2023;
```

Look for `partitions: p2023` in the EXPLAIN output, confirming pruning is working.

## Adding a New Partition

```sql
ALTER TABLE sales
ADD PARTITION (PARTITION p2025 VALUES LESS THAN (2026));
```

Note: You must first remove or reorganize `p_future` if it already covers the new range.

## Reorganizing Partitions

To add a 2025 partition and keep `p_future`:

```sql
ALTER TABLE sales REORGANIZE PARTITION p_future INTO (
  PARTITION p2025 VALUES LESS THAN (2026),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

## Dropping Old Partitions

One of the biggest benefits of RANGE partitioning is fast data deletion by dropping entire partitions:

```sql
ALTER TABLE sales DROP PARTITION p2021;
```

This is far faster than `DELETE FROM sales WHERE YEAR(sale_date) = 2021` because no row-by-row deletion occurs.

## Limitations

- The partition expression must return an integer (use `YEAR()`, `TO_DAYS()`, or `UNIX_TIMESTAMP()` for dates, or use `RANGE COLUMNS`)
- All partitioned tables must include the partition key in any PRIMARY KEY or UNIQUE KEY
- A maximum of 8192 partitions per table

## Summary

RANGE partitioning in MySQL divides a table into partitions by contiguous value ranges, making it ideal for time-series and date-based data. It enables fast partition pruning during queries, efficient bulk deletion by dropping whole partitions, and easier data lifecycle management. Use `RANGE COLUMNS` when you need to partition on date types or multiple columns directly.
