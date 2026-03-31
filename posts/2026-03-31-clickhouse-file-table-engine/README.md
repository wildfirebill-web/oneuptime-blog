# How to Use File Table Engine in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, File Engine, Table Engine, CSV, Parquet, Local File, SQL

Description: Learn how to use the ClickHouse File table engine to read and write local files (CSV, Parquet, TSV, JSON) as ClickHouse tables for import, export, and ETL.

---

The `File` table engine in ClickHouse treats a local file on the ClickHouse server as a table. You can SELECT from it, INSERT into it, and use it in JOINs - making it ideal for file-based data import/export, ETL staging, and integration with file-producing tools.

## Syntax

```sql
ENGINE = File(format)
```

The file is stored at: `{user_files_path}/{table_name}.{format_extension}`

The default `user_files_path` is `/var/lib/clickhouse/user_files/`.

## Creating a File Table (CSV)

```sql
CREATE TABLE csv_import (
    id     UInt32,
    name   String,
    amount Float64,
    date   Date
) ENGINE = File(CSV);
```

This creates a table backed by `/var/lib/clickhouse/user_files/csv_import.csv`.

## Inserting Data (Exporting to File)

```sql
INSERT INTO csv_import
SELECT order_id, customer_name, total_amount, order_date
FROM orders
WHERE order_date = today() - 1;
```

The data is written to the CSV file.

## Reading Data (Importing from File)

Place a file at `/var/lib/clickhouse/user_files/csv_import.csv`:

```bash
sudo cp /tmp/orders.csv /var/lib/clickhouse/user_files/csv_import.csv
sudo chown clickhouse:clickhouse /var/lib/clickhouse/user_files/csv_import.csv
```

Then query it:

```sql
SELECT * FROM csv_import LIMIT 10;
```

## Supported Formats

```text
Format         Extension  Description
CSV            .csv       Comma-separated with quoting
TSV            .tsv       Tab-separated
JSONEachRow    .jsonl     One JSON object per line
Parquet        .parquet   Columnar binary format
ORC            .orc       Optimized row columnar
Native         .bin       ClickHouse native binary
```

## Creating a Parquet File Table

```sql
CREATE TABLE parquet_export (
    ts       DateTime,
    user_id  UInt64,
    event    String,
    value    Float64
) ENGINE = File(Parquet);
```

Export:

```sql
INSERT INTO parquet_export
SELECT ts, user_id, event, value
FROM events
WHERE toDate(ts) = yesterday();
```

The Parquet file is at `/var/lib/clickhouse/user_files/parquet_export.parquet`.

## Specifying a Custom File Path

You can specify a path relative to `user_files_path`:

```sql
CREATE TABLE my_data ENGINE = File(CSV) AS SELECT 1;

-- Custom path syntax using clickhouse-local
-- SELECT * FROM file('/var/lib/clickhouse/user_files/data.csv', CSV, 'id UInt32, name String');
```

## Using file() Table Function for One-Off Reads

For ad hoc file reads without creating a persistent table, use the `file()` function:

```sql
SELECT *
FROM file('/var/lib/clickhouse/user_files/orders.csv', CSV, 'id UInt32, name String, amount Float64');
```

## Data Pipeline: Import, Transform, Export

```sql
-- 1. Create staging table backed by input CSV
CREATE TABLE input_staging (raw_line String) ENGINE = File(LineAsString);

-- 2. Query and transform
SELECT
    toUInt32(extractAll(raw_line, '(\\d+)')[1]) AS id,
    trim(extract(raw_line, ',"([^"]+)"')) AS name
FROM input_staging;
```

## Limitations

- File engine does not support concurrent writes - only one writer at a time.
- No partitioning, indexing, or TTL.
- The file must be on the ClickHouse server's local filesystem.
- Large files are read with full scans.

## Summary

The File table engine lets you read and write local files (CSV, Parquet, TSV, JSON) as ClickHouse tables. Use it for file-based imports, exports, and ETL staging. For ad hoc one-off file reads, use the `file()` table function. For production ingestion pipelines, migrate data into MergeTree tables for indexing and query performance.
