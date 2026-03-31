# How to Use clickhouse-local for Local File Processing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, clickhouse-local, File Processing, CLI, ETL, Data Transformation

Description: Use clickhouse-local to run ClickHouse SQL queries directly against local files without a server, enabling powerful data transformation and analysis pipelines.

---

`clickhouse-local` is a standalone tool that brings ClickHouse's SQL engine to local file processing - without requiring a running server. It can read CSV, JSON, Parquet, ORC, and many other formats, run full SQL queries including JOINs and aggregations, and write results in any ClickHouse output format.

## What is clickhouse-local?

`clickhouse-local` runs as a single process with an embedded ClickHouse engine. It is ideal for:

- One-time data transformations on files
- ETL preprocessing before loading into ClickHouse
- Analyzing log files or exports locally
- Building data pipelines in shell scripts

## Installation

`clickhouse-local` ships with the ClickHouse binary. Download it directly:

```bash
curl https://clickhouse.com/ | sh
./clickhouse local --version
```

## Basic Usage: Query a CSV File

```bash
clickhouse-local \
    --query "SELECT count(), min(ts), max(ts) FROM file('events.csv', CSV, 'id UInt64, ts DateTime, action String')"
```

## Auto-Detecting Schema

Use `DESCRIBE` to infer the schema from a file:

```bash
clickhouse-local \
    --query "DESCRIBE TABLE file('events.jsonl', JSONEachRow)"
```

## Processing JSON Lines

```bash
clickhouse-local \
    --query "
    SELECT
        toDate(ts)   AS day,
        action,
        count()      AS cnt
    FROM file('events.jsonl', JSONEachRow)
    GROUP BY day, action
    ORDER BY day DESC, cnt DESC
    FORMAT PrettyCompact
"
```

## Joining Multiple Files

```bash
clickhouse-local --query "
SELECT
    e.action,
    u.name,
    count() AS events
FROM file('events.csv', CSV, 'user_id UInt32, ts DateTime, action String') AS e
JOIN file('users.csv', CSV, 'user_id UInt32, name String') AS u
  ON e.user_id = u.user_id
GROUP BY e.action, u.name
ORDER BY events DESC
LIMIT 20
"
```

## Converting File Formats

Convert CSV to Parquet:

```bash
clickhouse-local \
    --query "
    SELECT *
    FROM file('input.csv', CSV, 'id UInt64, ts DateTime, value Float64')
    INTO OUTFILE 'output.parquet'
    FORMAT Parquet
"
```

Convert JSON to CSV:

```bash
clickhouse-local \
    --query "SELECT * FROM file('events.jsonl', JSONEachRow) FORMAT CSV" \
    > events.csv
```

## Filtering and Transforming Data

```bash
clickhouse-local --query "
SELECT
    id,
    toUnixTimestamp(parseDateTime(ts, '%Y-%m-%dT%H:%i:%s')) AS ts_unix,
    upper(action)                                            AS action,
    if(revenue > 0, revenue, 0)                              AS revenue
FROM file('raw_events.csv', CSVWithNames)
WHERE action != 'bot'
FORMAT JSONEachRow
" > cleaned_events.jsonl
```

## Processing Gzipped Files

```bash
clickhouse-local \
    --query "SELECT count() FROM file('events.csv.gz', CSV, 'id UInt64, ts DateTime')"
```

ClickHouse auto-detects gzip compression from the `.gz` extension.

## Passing Files via stdin

```bash
cat events.csv | clickhouse-local \
    --query "SELECT count() FROM table" \
    --input-format CSV \
    --structure "id UInt64, ts DateTime, action String"
```

## Setting Memory Limits

For very large files, set a memory limit to prevent OOM:

```bash
clickhouse-local \
    --max_memory_usage 4000000000 \
    --query "SELECT action, count() FROM file('huge.csv', CSV, 'action String, ...')"
```

## Summary

`clickhouse-local` brings the full power of ClickHouse SQL to local file processing without requiring a server. Use it for quick file analysis, format conversion, complex multi-file JOINs, and preprocessing steps in data pipelines. It supports the same SQL syntax, functions, and formats as the server, making it the most powerful tool for ad-hoc data work.
