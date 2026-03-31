# How to Use LIST Partitioning in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Partitioning, List Partitioning, Database Design, Performance

Description: Learn how to implement LIST partitioning in MySQL to group rows by discrete column values, with syntax, examples, and partition management operations.

---

## What Is LIST Partitioning?

LIST partitioning divides a table based on discrete column values rather than ranges. Each partition is assigned a specific set of values. Unlike RANGE partitioning, LIST partitioning is ideal when the column contains a finite set of known values such as region codes, status flags, or category IDs.

## Basic Syntax

```sql
CREATE TABLE table_name (
  column_definitions
)
PARTITION BY LIST (column_expr) (
  PARTITION p0 VALUES IN (value1, value2, ...),
  PARTITION p1 VALUES IN (value3, value4, ...),
  ...
);
```

## Example: Partitioning by Region

```sql
CREATE TABLE orders (
  id INT NOT NULL AUTO_INCREMENT,
  region_id INT NOT NULL,
  customer_id INT NOT NULL,
  order_date DATE NOT NULL,
  total DECIMAL(10,2),
  PRIMARY KEY (id, region_id)
)
PARTITION BY LIST (region_id) (
  PARTITION p_north   VALUES IN (1, 2, 3),
  PARTITION p_south   VALUES IN (4, 5, 6),
  PARTITION p_east    VALUES IN (7, 8, 9),
  PARTITION p_west    VALUES IN (10, 11, 12)
);
```

## Example: Partitioning by Status

```sql
CREATE TABLE tickets (
  id INT NOT NULL AUTO_INCREMENT,
  status TINYINT NOT NULL,
  created_at DATETIME,
  title VARCHAR(255),
  PRIMARY KEY (id, status)
)
PARTITION BY LIST (status) (
  PARTITION p_open     VALUES IN (1),
  PARTITION p_in_prog  VALUES IN (2),
  PARTITION p_resolved VALUES IN (3, 4),
  PARTITION p_closed   VALUES IN (5, 6)
);
```

## Using LIST COLUMNS

`LIST COLUMNS` supports partitioning on string or date columns:

```sql
CREATE TABLE customers (
  id INT NOT NULL,
  country CHAR(2) NOT NULL,
  name VARCHAR(100),
  PRIMARY KEY (id, country)
)
PARTITION BY LIST COLUMNS (country) (
  PARTITION p_americas VALUES IN ('US', 'CA', 'MX', 'BR'),
  PARTITION p_europe   VALUES IN ('GB', 'DE', 'FR', 'IT', 'ES'),
  PARTITION p_apac     VALUES IN ('JP', 'CN', 'AU', 'IN', 'SG')
);
```

## Inserting Data

```sql
INSERT INTO orders (region_id, customer_id, order_date, total) VALUES
(1, 101, '2024-01-15', 250.00),
(5, 102, '2024-01-20', 400.00),
(8, 103, '2024-02-01', 150.00);
```

## Important: Unmatched Values Cause Errors

Unlike RANGE partitioning, there is no `MAXVALUE` fallback in LIST partitioning. If you insert a value not covered by any partition, MySQL raises an error:

```sql
-- This will fail if region_id = 20 is not in any partition
INSERT INTO orders (region_id, customer_id, order_date, total)
VALUES (20, 104, '2024-03-01', 500.00);
-- ERROR 1526: Table has no partition for value 20
```

## Verifying Partition Contents

```sql
SELECT
  partition_name,
  partition_description,
  table_rows
FROM information_schema.partitions
WHERE table_name = 'orders'
  AND table_schema = 'mydb';
```

## Querying a Specific Partition

```sql
SELECT * FROM orders PARTITION (p_north);
```

## Adding a Partition

```sql
ALTER TABLE orders
ADD PARTITION (PARTITION p_intl VALUES IN (13, 14, 15));
```

## Reorganizing a Partition

Split one partition into two:

```sql
ALTER TABLE orders REORGANIZE PARTITION p_south INTO (
  PARTITION p_south_a VALUES IN (4, 5),
  PARTITION p_south_b VALUES IN (6)
);
```

## Dropping a Partition

```sql
ALTER TABLE orders DROP PARTITION p_intl;
```

Dropping a partition removes all rows within it, so back up data first if needed.

## Merging Partitions

Merge multiple partitions back into one using REORGANIZE:

```sql
ALTER TABLE orders REORGANIZE PARTITION p_south_a, p_south_b INTO (
  PARTITION p_south VALUES IN (4, 5, 6)
);
```

## Checking Which Partition a Row Goes Into

```sql
EXPLAIN SELECT * FROM orders WHERE region_id = 5;
```

The `partitions` column in the EXPLAIN output shows which partition(s) will be scanned.

## Partition Pruning

MySQL automatically prunes irrelevant partitions when the WHERE clause filters on the partition key:

```sql
-- Only p_north is scanned
SELECT * FROM orders WHERE region_id IN (1, 2, 3);
```

## Summary

LIST partitioning in MySQL routes rows to partitions based on discrete, predefined column values. It is ideal for categorical data like region codes, status values, or country codes. Use `LIST COLUMNS` for string-based partitioning. Unlike RANGE partitioning, all possible values must be explicitly assigned to a partition or inserts will fail, so plan the value set carefully.
