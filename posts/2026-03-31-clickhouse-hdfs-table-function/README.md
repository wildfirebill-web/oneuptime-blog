# How to Use hdfs() Table Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, hdfs(), Table Function, HDFS, Hadoop, Analytics, SQL

Description: Learn how to use the hdfs() table function in ClickHouse to read files from Hadoop HDFS directly in SQL queries for analytics and data migration.

---

The `hdfs()` table function in ClickHouse lets you read files stored in Hadoop Distributed File System (HDFS) directly from SQL queries, without creating a permanent table. It supports glob patterns for multi-file reads and accepts formats like Parquet, CSV, JSONEachRow, and ORC - making it ideal for ad hoc analytics over Hadoop data lakes.

## Syntax

```sql
hdfs('hdfs_uri', 'format')
hdfs('hdfs_uri', 'format', 'structure')
```

- `hdfs_uri`: HDFS URI, e.g., `hdfs://namenode:9000/path/to/file.parquet`
- `format`: ClickHouse input format (Parquet, CSV, TSV, JSONEachRow, ORC, etc.)
- `structure`: optional column definitions; if omitted, ClickHouse infers the schema

## Reading a Single Parquet File

```sql
SELECT *
FROM hdfs('hdfs://namenode:9000/data/sales/2026-01.parquet', 'Parquet')
LIMIT 10;
```

## Reading Multiple Files with Glob Patterns

```sql
-- Read all Parquet files for Q1 2026
SELECT
    product_id,
    sum(revenue) AS total_revenue
FROM hdfs('hdfs://namenode:9000/data/sales/2026-0{1,2,3}-*.parquet', 'Parquet')
GROUP BY product_id
ORDER BY total_revenue DESC;
```

```sql
-- Wildcard - read all CSV files in a directory
SELECT count() FROM hdfs('hdfs://namenode:9000/data/events/*.csv', 'CSV');
```

## Reading JSON Lines Files

```sql
SELECT
    event_type,
    count() AS cnt
FROM hdfs(
    'hdfs://namenode:9000/logs/events-*.jsonl',
    'JSONEachRow',
    'event_type String, user_id UInt64, ts DateTime'
)
GROUP BY event_type
ORDER BY cnt DESC;
```

## Inspecting Schema of a Parquet File

```sql
DESCRIBE TABLE hdfs('hdfs://namenode:9000/data/orders/orders-2026.parquet', 'Parquet');
```

## Loading HDFS Data into ClickHouse

```sql
INSERT INTO orders (order_id, customer_id, amount, created_at)
SELECT order_id, customer_id, amount, created_at
FROM hdfs('hdfs://namenode:9000/data/orders/*.parquet', 'Parquet');
```

## Reading ORC Files

```sql
SELECT
    region,
    sum(sales) AS total_sales
FROM hdfs('hdfs://namenode:9000/warehouse/sales/region=*.orc', 'ORC')
GROUP BY region;
```

## Configuration

Ensure your ClickHouse server can reach the HDFS namenode. Configure Kerberos or simple auth in `config.xml` if needed:

```text
<hdfs>
    <hadoop_kerberos_keytab>/etc/security/keytabs/clickhouse.keytab</hadoop_kerberos_keytab>
    <hadoop_kerberos_principal>clickhouse/host@REALM</hadoop_kerberos_principal>
    <hadoop_security_authentication>kerberos</hadoop_security_authentication>
</hdfs>
```

For simple auth without Kerberos:

```text
<hdfs>
    <hadoop_security_authentication>simple</hadoop_security_authentication>
</hdfs>
```

## Filtering at Read Time

Push predicates into the query to reduce the data ClickHouse loads:

```sql
SELECT order_id, amount
FROM hdfs('hdfs://namenode:9000/data/orders/*.parquet', 'Parquet')
WHERE toDate(created_at) = '2026-03-01'
  AND amount > 100.0;
```

## Summary

The `hdfs()` table function gives ClickHouse direct read access to HDFS files in any supported format. Use glob patterns to query across multiple files, push WHERE filters to minimize I/O, and use `INSERT INTO ... SELECT ... FROM hdfs(...)` to migrate Hadoop data into ClickHouse. For regular or high-frequency access, the HDFS table engine with a named table is more appropriate than repeated function calls.
