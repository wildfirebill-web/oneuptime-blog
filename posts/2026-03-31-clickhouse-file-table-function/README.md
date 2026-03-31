# How to Use file() Table Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Table Function, File, SQL, Database, ETL

Description: Learn how to use the file() table function in ClickHouse to read and write local files in formats like CSV, TSV, JSON, and Parquet directly from SQL queries.

---

The `file()` table function in ClickHouse lets you read data from local files on the server's filesystem and use the results like any ordinary table. You can query, filter, join, and aggregate file data with standard SQL, and you can also write query results back to files.

## What Is the file() Table Function?

`file()` opens a file on the ClickHouse server's filesystem and presents its rows as a virtual table. No import step is needed - you query the file directly.

```sql
SELECT *
FROM file('/var/lib/clickhouse/user_files/sales.csv', 'CSV', 'date Date, product String, amount Float64')
LIMIT 10;
```

## Basic Syntax

```sql
file(path, format, structure)
```

| Parameter   | Description |
|-------------|-------------|
| `path`      | Absolute path on the ClickHouse server, or a glob pattern |
| `format`    | Input format: `CSV`, `TSV`, `JSONEachRow`, `Parquet`, `ORC`, etc. |
| `structure` | Column definitions as `'col_name type, ...'` |

## Security: The user_files Directory

By default, ClickHouse restricts `file()` access to the `user_files` directory (typically `/var/lib/clickhouse/user_files/`). You can configure this in `config.xml`:

```xml
<user_files_path>/var/lib/clickhouse/user_files/</user_files_path>
```

Place your files there before querying them, or adjust the path in your config.

## Reading a CSV File

```sql
-- sales.csv contains: 2026-01-01,Laptop,1299.99
SELECT
    date,
    product,
    amount,
    toStartOfMonth(date) AS month
FROM file(
    '/var/lib/clickhouse/user_files/sales.csv',
    'CSV',
    'date Date, product String, amount Float64'
)
WHERE amount > 500
ORDER BY date;
```

## Reading CSV with a Header Row

Use `CSVWithNames` to automatically use the first row as column names:

```sql
SELECT *
FROM file(
    '/var/lib/clickhouse/user_files/sales_with_header.csv',
    'CSVWithNames',
    'date Date, product String, amount Float64'
);
```

## Reading TSV Files

```sql
SELECT count(), avg(response_ms)
FROM file(
    '/var/lib/clickhouse/user_files/access.tsv',
    'TSV',
    'ts DateTime, path String, status UInt16, response_ms UInt32'
)
WHERE status >= 500;
```

## Reading JSON (JSONEachRow)

Each line must be a valid JSON object:

```sql
-- events.jsonl: {"user_id":1,"event":"click","ts":"2026-03-01 12:00:00"}
SELECT
    user_id,
    event,
    toDate(ts) AS event_date
FROM file(
    '/var/lib/clickhouse/user_files/events.jsonl',
    'JSONEachRow',
    'user_id UInt32, event String, ts DateTime'
)
GROUP BY user_id, event, event_date
ORDER BY user_id;
```

## Reading Parquet Files

```sql
SELECT *
FROM file(
    '/var/lib/clickhouse/user_files/metrics.parquet',
    'Parquet',
    'host String, cpu Float32, memory Float32, recorded_at DateTime'
)
WHERE cpu > 90
LIMIT 100;
```

## Glob Patterns: Reading Multiple Files at Once

`file()` accepts glob patterns, so you can process many files in a single query:

```sql
-- Read all CSV files in a directory
SELECT count(), sum(amount)
FROM file(
    '/var/lib/clickhouse/user_files/sales_*.csv',
    'CSV',
    'date Date, product String, amount Float64'
);

-- Read files matching a date range pattern
SELECT *
FROM file(
    '/var/lib/clickhouse/user_files/logs/2026-01-{01..31}.csv',
    'CSV',
    'ts DateTime, level String, message String'
);
```

## Writing Results to a File

Use `INSERT INTO FUNCTION file(...)` to write query results to disk:

```sql
INSERT INTO FUNCTION file(
    '/var/lib/clickhouse/user_files/monthly_summary.csv',
    'CSV',
    'month Date, product String, total_sales Float64'
)
SELECT
    toStartOfMonth(date) AS month,
    product,
    sum(amount)          AS total_sales
FROM sales
GROUP BY month, product
ORDER BY month, product;
```

## Loading File Data into a Table

The standard pattern for importing file data into a permanent table:

```sql
CREATE TABLE sales
(
    date    Date,
    product String,
    amount  Float64
)
ENGINE = MergeTree()
ORDER BY (date, product);

INSERT INTO sales
SELECT *
FROM file(
    '/var/lib/clickhouse/user_files/sales.csv',
    'CSV',
    'date Date, product String, amount Float64'
);
```

## Auto-Detecting Format and Schema

For some formats, ClickHouse can infer the schema automatically. With the `DESCRIBE` statement you can preview what ClickHouse detects:

```sql
DESCRIBE file('/var/lib/clickhouse/user_files/metrics.parquet', 'Parquet');
```

Then use the detected schema in your actual query.

## Using file() in a JOIN

```sql
SELECT
    o.order_id,
    o.amount,
    r.region_name
FROM orders AS o
JOIN file(
    '/var/lib/clickhouse/user_files/regions.csv',
    'CSV',
    'region_code String, region_name String'
) AS r ON o.region_code = r.region_code;
```

## Performance Tips

- For large files, load data into a MergeTree table first. Direct querying of files bypasses ClickHouse's indexing and part pruning.
- Use Parquet or ORC for columnar reads - ClickHouse can push column projections into these formats and skip unneeded columns.
- For many small files, batching them into one large file before querying reduces overhead.
- `file()` reads are single-threaded per file by default. Use `max_threads` to allow parallelism across multiple files matched by a glob.

## Common Errors and Fixes

**File not found:** Ensure the file is inside `user_files_path` or update `config.xml`.

```bash
sudo cp sales.csv /var/lib/clickhouse/user_files/
sudo chown clickhouse:clickhouse /var/lib/clickhouse/user_files/sales.csv
```

**Format mismatch:** If your CSV uses a semicolon delimiter, use `format_csv_delimiter`:

```sql
SET format_csv_delimiter = ';';

SELECT *
FROM file('/var/lib/clickhouse/user_files/eu_sales.csv', 'CSV', 'date Date, product String, amount Float64');
```

## Summary

The `file()` table function makes ClickHouse a powerful tool for ad-hoc file analysis. Key points:

- Read CSV, TSV, JSON, Parquet, ORC, and many other formats directly.
- Use glob patterns to process multiple files at once.
- Write query results back to disk with `INSERT INTO FUNCTION file(...)`.
- Load frequently queried file data into MergeTree tables for full index performance.
