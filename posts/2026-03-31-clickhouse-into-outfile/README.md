# How to Use INTO OUTFILE to Export ClickHouse Query Results

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, SELECT, INTO OUTFILE, Export, Data Export

Description: Export ClickHouse query results directly to a file using INTO OUTFILE with CSV, TSV, Parquet, and other formats including compression support.

---

ClickHouse lets you write query results straight to a file on the server by appending `INTO OUTFILE` to any `SELECT` statement. This is useful for data pipelines, backups, and hand-offs to external tools - no intermediate ETL step required.

## Basic Syntax

The minimal form names the output file and lets ClickHouse infer the format from the extension:

```sql
SELECT
    user_id,
    event_name,
    toDate(timestamp) AS event_date
FROM events
INTO OUTFILE '/var/lib/clickhouse/user_files/events_export.csv'
FORMAT CSV;
```

The file path must reside inside the directory specified by `user_files_path` in `config.xml` (default `/var/lib/clickhouse/user_files/`). Absolute paths outside that directory are rejected unless the server is configured otherwise.

## Supported Format Options

ClickHouse supports dozens of output formats. The most commonly used ones for file export are:

```sql
-- Comma-separated with header
SELECT id, name, value
FROM my_table
INTO OUTFILE '/var/lib/clickhouse/user_files/out.csv'
FORMAT CSVWithNames;

-- Tab-separated with header
SELECT id, name, value
FROM my_table
INTO OUTFILE '/var/lib/clickhouse/user_files/out.tsv'
FORMAT TabSeparatedWithNames;

-- Columnar Parquet (great for analytics pipelines)
SELECT id, name, value
FROM my_table
INTO OUTFILE '/var/lib/clickhouse/user_files/out.parquet'
FORMAT Parquet;

-- JSON Lines (one JSON object per row)
SELECT id, name, value
FROM my_table
INTO OUTFILE '/var/lib/clickhouse/user_files/out.jsonl'
FORMAT JSONEachRow;

-- Native ClickHouse binary format (fastest for ClickHouse-to-ClickHouse transfers)
SELECT id, name, value
FROM my_table
INTO OUTFILE '/var/lib/clickhouse/user_files/out.native'
FORMAT Native;
```

## Compression

Append a compression codec after the format to produce compressed files without a separate step:

```sql
-- gzip-compressed CSV
SELECT *
FROM orders
WHERE toYear(created_at) = 2025
INTO OUTFILE '/var/lib/clickhouse/user_files/orders_2025.csv.gz'
FORMAT CSV
COMPRESSION 'gzip';

-- LZ4-compressed Parquet
SELECT *
FROM events
INTO OUTFILE '/var/lib/clickhouse/user_files/events.parquet.lz4'
FORMAT Parquet
COMPRESSION 'lz4';

-- zstd-compressed TSV
SELECT *
FROM logs
INTO OUTFILE '/var/lib/clickhouse/user_files/logs.tsv.zst'
FORMAT TabSeparated
COMPRESSION 'zstd';
```

Supported compression codecs include `gzip`, `gz`, `brotli`, `br`, `xz`, `zstd`, `zst`, and `lz4`.

## Overwriting Existing Files

By default ClickHouse raises an error if the output file already exists. Use `TRUNCATE` or `APPEND` to control the behavior:

```sql
-- Overwrite the file if it exists
SELECT *
FROM events
INTO OUTFILE '/var/lib/clickhouse/user_files/daily_export.csv'
FORMAT CSVWithNames
SETTINGS engine_file_truncate_on_insert = 1;

-- Append to the file instead of overwriting
SELECT *
FROM events
WHERE toDate(timestamp) = today()
INTO OUTFILE '/var/lib/clickhouse/user_files/rolling_export.csv'
FORMAT CSV
APPEND;
```

## Exporting from clickhouse-client

When using the `clickhouse-client` CLI you can also redirect output to a local file without server-side file permissions:

```bash
# Export via clickhouse-client (file lands on the client machine)
clickhouse-client \
  --query "SELECT * FROM events FORMAT CSVWithNames" \
  > /tmp/events_export.csv

# Compressed export via CLI
clickhouse-client \
  --query "SELECT * FROM events FORMAT CSVWithNames" \
  | gzip > /tmp/events_export.csv.gz

# Using INTO OUTFILE through clickhouse-client (server-side path)
clickhouse-client \
  --query "SELECT * FROM events INTO OUTFILE '/var/lib/clickhouse/user_files/events.csv' FORMAT CSV"
```

## Practical Export Pipeline Example

```sql
-- Export aggregated daily metrics partitioned by month
SELECT
    toDate(timestamp)          AS day,
    country,
    count()                    AS total_events,
    countIf(status = 'error')  AS error_events,
    avg(duration_ms)           AS avg_duration
FROM events
WHERE toYYYYMM(timestamp) = 202501
GROUP BY day, country
ORDER BY day, country
INTO OUTFILE '/var/lib/clickhouse/user_files/metrics_202501.parquet'
FORMAT Parquet
COMPRESSION 'zstd';
```

## Summary

`INTO OUTFILE` turns a `SELECT` query into a direct file export, eliminating the need for client-side redirection or a separate copy step. You can choose from CSV, TSV, Parquet, JSONEachRow, Native, and many other formats, and optionally compress the output in the same statement. Keep file paths within the configured `user_files_path` directory, and use `APPEND` or `engine_file_truncate_on_insert` when you need to handle pre-existing files.
