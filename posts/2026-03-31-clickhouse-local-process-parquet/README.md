# How to Process Parquet Files with clickhouse-local

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, clickhouse-local, Parquet, Columnar, Data Processing

Description: Learn how to process Parquet files with clickhouse-local to run SQL queries on columnar data without a database server or import step.

---

## Why Process Parquet with clickhouse-local?

Parquet is a columnar format widely used in data lakes, Spark pipelines, and S3 exports. `clickhouse-local` reads Parquet files natively and executes SQL queries with full column pruning - only the columns you SELECT are read from disk.

## Reading a Parquet File

```bash
clickhouse local --query "
SELECT * FROM file('data.parquet', Parquet) LIMIT 5
"
```

## Inspecting Parquet Schema

```bash
clickhouse local --query "DESCRIBE TABLE file('data.parquet', Parquet)"
```

Output:

```text
name         type
user_id      UInt64
email        String
signup_date  Date
revenue      Float64
```

## Efficient Column Pruning

Only `user_id` and `revenue` columns are read from the Parquet file:

```bash
clickhouse local --query "
SELECT user_id, sum(revenue) AS total
FROM file('data.parquet', Parquet)
GROUP BY user_id
ORDER BY total DESC
LIMIT 10
"
```

## Filtering on Partitioned Parquet

When Parquet files are partitioned by date (Hive-style), use glob patterns:

```bash
clickhouse local --query "
SELECT count(), sum(amount)
FROM file('/data/orders/year=2024/month=*/part-*.parquet', Parquet)
WHERE amount > 100
"
```

## Converting Parquet to CSV

```bash
clickhouse local \
  --query "SELECT * FROM file('data.parquet', Parquet)" \
  --format CSVWithNames > data.csv
```

## Converting CSV to Parquet

```bash
clickhouse local \
  --query "SELECT * FROM file('data.csv', CSVWithNames)" \
  --format Parquet > data.parquet
```

## Joining Parquet Files

```bash
clickhouse local --query "
SELECT
    o.order_id,
    o.amount,
    c.name AS customer_name,
    c.country
FROM file('orders.parquet', Parquet) o
JOIN file('customers.parquet', Parquet) c ON o.customer_id = c.id
WHERE o.amount > 500
ORDER BY o.amount DESC
LIMIT 20
"
```

## Aggregating Large Parquet Exports

```bash
clickhouse local --query "
SELECT
    toStartOfMonth(order_date) AS month,
    category,
    sum(revenue) AS total_revenue,
    count() AS order_count
FROM file('/exports/orders_*.parquet', Parquet)
GROUP BY month, category
ORDER BY month, total_revenue DESC
" --format PrettyCompact
```

## Summary

`clickhouse-local` processes Parquet files natively with full column pruning, glob-based multi-file queries, and format conversion. It supports reading Hive-partitioned directory layouts, making it a fast tool for exploring data lake exports without spinning up a cluster.
