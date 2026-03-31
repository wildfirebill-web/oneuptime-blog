# How to Join Multiple Files with clickhouse-local

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, clickhouse-local, Join, Multi-File, Data Analysis

Description: Learn how to join multiple files in different formats with clickhouse-local to combine CSV, JSON, and Parquet data sources using SQL joins.

---

## Joining Files with clickhouse-local

One of the most powerful features of `clickhouse-local` is joining data from multiple files in different formats - CSV, JSON, Parquet - in a single SQL query without loading them into a server first.

## Basic CSV-to-CSV Join

```bash
clickhouse local --query "
SELECT
    o.order_id,
    o.amount,
    c.customer_name,
    c.email
FROM file('orders.csv', CSVWithNames) o
JOIN file('customers.csv', CSVWithNames) c ON o.customer_id = c.id
ORDER BY o.amount DESC
LIMIT 20
"
```

## Cross-Format Join (CSV + JSON)

```bash
clickhouse local --query "
SELECT
    o.order_id,
    o.amount,
    p.name AS product_name,
    p.category
FROM file('orders.csv', CSVWithNames) o
JOIN file('products.ndjson', JSONEachRow) p ON o.product_id = p.id
WHERE p.category = 'Electronics'
ORDER BY o.amount DESC
"
```

## Three-Way Join

```bash
clickhouse local --query "
SELECT
    o.order_id,
    c.customer_name,
    p.product_name,
    o.quantity,
    o.unit_price * o.quantity AS line_total
FROM file('orders.csv', CSVWithNames) o
JOIN file('customers.parquet', Parquet) c ON o.customer_id = c.id
JOIN file('products.csv', CSVWithNames) p ON o.product_id = p.id
ORDER BY line_total DESC
LIMIT 50
"
```

## LEFT JOIN for Finding Missing Records

Identify orders with no matching customer:

```bash
clickhouse local --query "
SELECT o.order_id, o.customer_id, o.amount
FROM file('orders.csv', CSVWithNames) o
LEFT JOIN file('customers.csv', CSVWithNames) c ON o.customer_id = c.id
WHERE c.id IS NULL
ORDER BY o.amount DESC
"
```

## Anti-Join for Data Quality

Find customers with no orders:

```bash
clickhouse local --query "
SELECT c.id, c.customer_name, c.email
FROM file('customers.csv', CSVWithNames) c
LEFT ANTI JOIN file('orders.csv', CSVWithNames) o ON c.id = o.customer_id
ORDER BY c.id
"
```

## Joining with Inline Data

Use VALUES as an inline lookup table:

```bash
clickhouse local --query "
SELECT
    o.order_id,
    o.status_code,
    s.description AS status
FROM file('orders.csv', CSVWithNames) o
JOIN (
    VALUES
        (1, 'Pending'),
        (2, 'Processing'),
        (3, 'Shipped'),
        (4, 'Delivered'),
        (5, 'Cancelled')
) AS s (code, description) ON o.status_code = s.code
"
```

## Performance Tip: Smaller Table on the Right

```bash
clickhouse local --query "
-- Put the larger file on the left, smaller lookup on the right
SELECT l.*, r.category
FROM file('large_events.parquet', Parquet) l
JOIN file('small_lookup.csv', CSVWithNames) r ON l.product_id = r.id
" --max_memory_usage 4000000000
```

## Summary

`clickhouse-local` can join files of any supported format - CSV, JSON, Parquet, TSV - in a single SQL query. LEFT JOIN detects missing records for data quality checks, and placing the smaller file on the right side of a JOIN improves performance for large file combinations.
