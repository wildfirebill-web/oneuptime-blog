# How to Process CSV Files with clickhouse-local

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, clickhouse-local, CSV, Data Processing, CLI

Description: Learn how to process CSV files with clickhouse-local including filtering, aggregating, joining, and transforming CSV data using SQL from the command line.

---

## Processing CSV with clickhouse-local

`clickhouse-local` handles CSV files natively without any import step. You can filter, transform, join, and aggregate CSV data using full ClickHouse SQL directly from the shell.

## Reading a CSV with Headers

```bash
clickhouse local --query "
SELECT * FROM file('orders.csv', CSVWithNames) LIMIT 5
"
```

## Reading a CSV without Headers

Provide the schema explicitly:

```bash
clickhouse local --query "
SELECT * FROM file('orders.csv', CSV, 'order_id UInt64, customer String, amount Float64, date Date')
LIMIT 5
"
```

## Filtering Rows

```bash
clickhouse local --query "
SELECT order_id, customer, amount
FROM file('orders.csv', CSVWithNames)
WHERE amount > 500
  AND toYear(date) = 2024
ORDER BY amount DESC
LIMIT 20
"
```

## Aggregating CSV Data

```bash
clickhouse local --query "
SELECT
    customer,
    count() AS orders,
    sum(amount) AS total,
    avg(amount) AS avg_order
FROM file('orders.csv', CSVWithNames)
GROUP BY customer
ORDER BY total DESC
LIMIT 10
"
```

## Handling Different CSV Delimiters

For tab-separated or semicolon-delimited files:

```bash
# Tab-separated
clickhouse local --query "SELECT * FROM file('data.tsv', TSVWithNames) LIMIT 5"

# Semicolon-delimited (use CustomSeparatedWithNames)
clickhouse local \
  --format_csv_delimiter=';' \
  --query "SELECT * FROM file('data.csv', CSVWithNames) LIMIT 5"
```

## Cleaning and Transforming Data

```bash
clickhouse local --query "
SELECT
    order_id,
    trim(customer) AS customer,
    replaceAll(sku, ' ', '_') AS sku_normalized,
    toFloat64OrZero(price_str) AS price,
    parseDateTimeBestEffort(date_str) AS order_date
FROM file('messy_orders.csv', CSV, 'order_id UInt64, customer String, sku String, price_str String, date_str String')
WHERE toFloat64OrZero(price_str) > 0
" --format CSV > cleaned_orders.csv
```

## Joining Two CSV Files

```bash
clickhouse local --query "
SELECT o.order_id, o.amount, c.region, c.tier
FROM file('orders.csv', CSVWithNames) o
JOIN file('customers.csv', CSVWithNames) c ON o.customer_id = c.id
ORDER BY o.amount DESC
LIMIT 20
"
```

## Deduplicating CSV Rows

```bash
clickhouse local --query "
SELECT DISTINCT *
FROM file('orders.csv', CSVWithNames)
ORDER BY order_id
" --format CSVWithNames > deduped_orders.csv
```

## Summary

`clickhouse-local` processes CSV files with full SQL including WHERE, GROUP BY, JOIN, and transformations. Use `CSVWithNames` for files with header rows and provide an explicit schema for headerless files. Pipe output to a new CSV for clean ETL pipelines.
