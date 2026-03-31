# How to Use COLUMNS Partitioning in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Partition, COLUMNS, Range, Performance

Description: Learn how to use RANGE COLUMNS and LIST COLUMNS partitioning in MySQL to partition on non-integer types like dates, strings, and multiple columns.

---

## What Is COLUMNS Partitioning?

Standard RANGE and LIST partitioning in MySQL requires an integer expression as the partition key. COLUMNS partitioning lifts this restriction, allowing you to partition directly on:

- Integer types (TINYINT through BIGINT)
- Date types (DATE, DATETIME)
- String types (CHAR, VARCHAR, BINARY, VARBINARY)
- Multiple columns simultaneously (composite key partitioning)

There are two variants: `RANGE COLUMNS` and `LIST COLUMNS`.

## RANGE COLUMNS on a DATE Column

Without COLUMNS, you need `TO_DAYS()` or `UNIX_TIMESTAMP()` to convert dates to integers. With RANGE COLUMNS, you use the date directly:

```sql
CREATE TABLE orders (
    order_id BIGINT NOT NULL,
    order_date DATE NOT NULL,
    customer_id INT NOT NULL,
    amount DECIMAL(10, 2),
    PRIMARY KEY (order_id, order_date)
)
PARTITION BY RANGE COLUMNS (order_date)
(
    PARTITION p2023 VALUES LESS THAN ('2024-01-01'),
    PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
    PARTITION p2025 VALUES LESS THAN ('2026-01-01'),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

## RANGE COLUMNS on a VARCHAR Column

Partition string data alphabetically:

```sql
CREATE TABLE customers (
    customer_id INT NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    first_name VARCHAR(100),
    email VARCHAR(200),
    PRIMARY KEY (customer_id, last_name)
)
PARTITION BY RANGE COLUMNS (last_name)
(
    PARTITION p_a_h VALUES LESS THAN ('I'),
    PARTITION p_i_p VALUES LESS THAN ('Q'),
    PARTITION p_q_z VALUES LESS THAN MAXVALUE
);
```

## LIST COLUMNS on a String

LIST COLUMNS lets you specify exact string values per partition:

```sql
CREATE TABLE sales_by_region (
    sale_id BIGINT NOT NULL,
    region VARCHAR(20) NOT NULL,
    amount DECIMAL(10, 2),
    PRIMARY KEY (sale_id, region)
)
PARTITION BY LIST COLUMNS (region)
(
    PARTITION p_americas VALUES IN ('US', 'CA', 'MX', 'BR'),
    PARTITION p_europe VALUES IN ('DE', 'FR', 'GB', 'IT', 'ES'),
    PARTITION p_apac VALUES IN ('JP', 'CN', 'AU', 'IN', 'SG')
);
```

## Multi-Column RANGE COLUMNS Partitioning

The most powerful feature of COLUMNS partitioning is composite partition keys. MySQL uses tuple comparison across all listed columns:

```sql
CREATE TABLE logs (
    log_id BIGINT NOT NULL,
    log_year SMALLINT NOT NULL,
    log_month TINYINT NOT NULL,
    message TEXT,
    PRIMARY KEY (log_id, log_year, log_month)
)
PARTITION BY RANGE COLUMNS (log_year, log_month)
(
    PARTITION p2024_h1 VALUES LESS THAN (2024, 7),
    PARTITION p2024_h2 VALUES LESS THAN (2025, 1),
    PARTITION p2025_h1 VALUES LESS THAN (2025, 7),
    PARTITION p2025_h2 VALUES LESS THAN (2026, 1),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

MySQL compares `(log_year, log_month)` tuples lexicographically - a row with `(2024, 8)` goes into `p2024_h2`.

## Check Partition Metadata

```sql
SELECT
    PARTITION_NAME,
    PARTITION_DESCRIPTION,
    PARTITION_METHOD,
    TABLE_ROWS
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'orders'
ORDER BY PARTITION_ORDINAL_POSITION;
```

## Differences from Regular RANGE/LIST

| Feature | RANGE/LIST | RANGE COLUMNS/LIST COLUMNS |
|---|---|---|
| Column types | Integer expression only | Date, String, Integer |
| Multiple columns | No | Yes |
| Expression support | Yes | No (direct column only) |

## Summary

RANGE COLUMNS and LIST COLUMNS partitioning eliminate the integer-only limitation of standard partitioning in MySQL. They let you partition directly on DATE and VARCHAR columns without conversions, and enable composite-column partition keys for more precise data routing. Use them whenever your natural partition key is not an integer.
