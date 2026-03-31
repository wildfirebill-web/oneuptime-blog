# How to Insert Data into ClickHouse Tables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Insert, Data Ingestion

Description: Learn how to insert data into ClickHouse tables using VALUES, batch inserts, the HTTP interface, and clickhouse-client with best practices.

---

ClickHouse is built for high-throughput writes, but getting the most out of it requires understanding how its INSERT mechanics work. Whether you are loading data from an application, a script, or a file, knowing the right approach - and the right batch size - makes the difference between a fast pipeline and a slow one. This post covers the core INSERT patterns every ClickHouse user should know.

## Basic INSERT INTO ... VALUES

The simplest way to insert data is the `INSERT INTO ... VALUES` syntax, identical to standard SQL.

```sql
CREATE TABLE events
(
    event_id   UInt64,
    event_name String,
    created_at DateTime
)
ENGINE = MergeTree()
ORDER BY (created_at, event_id);

INSERT INTO events (event_id, event_name, created_at) VALUES
    (1, 'page_view', '2026-03-31 10:00:00'),
    (2, 'click',     '2026-03-31 10:01:00'),
    (3, 'purchase',  '2026-03-31 10:02:00');
```

You can omit the column list when inserting values for every column in order:

```sql
INSERT INTO events VALUES
    (4, 'signup', '2026-03-31 10:03:00');
```

## Batch Inserts

ClickHouse writes each INSERT as a separate part on disk. Many small inserts create many small parts, which forces frequent background merges and degrades performance. The golden rule is: **batch your inserts**.

Aim for at least 1,000-10,000 rows per INSERT, ideally more. A single INSERT with 100,000 rows is far more efficient than 100,000 single-row inserts.

```sql
-- Good: one INSERT with many rows
INSERT INTO events (event_id, event_name, created_at) VALUES
    (10, 'page_view', '2026-03-31 11:00:00'),
    (11, 'click',     '2026-03-31 11:00:01'),
    -- ... thousands more rows
    (99999, 'logout', '2026-03-31 11:59:59');
```

## Inserting via the HTTP Interface

ClickHouse exposes an HTTP interface on port 8123 by default. You can POST data directly using `curl`.

```bash
# Insert using VALUES format
curl -X POST 'http://localhost:8123/?query=INSERT+INTO+events+FORMAT+Values' \
     --data-binary "(100, 'api_call', '2026-03-31 12:00:00'),(101, 'api_call', '2026-03-31 12:00:01')"
```

Pass credentials when authentication is enabled:

```bash
curl -X POST 'http://localhost:8123/' \
     -u myuser:mypassword \
     --data-binary "INSERT INTO events FORMAT Values (200, 'batch', '2026-03-31 13:00:00')"
```

## Inserting via clickhouse-client

`clickhouse-client` is the official CLI and supports several insert patterns.

### Interactive mode

```bash
clickhouse-client --query "INSERT INTO events VALUES (300, 'cli_insert', '2026-03-31 14:00:00')"
```

### Piping data from a file

```bash
# Insert from a TSV file
clickhouse-client --query "INSERT INTO events FORMAT TabSeparated" < events.tsv
```

### Heredoc for quick scripted inserts

```bash
clickhouse-client --multiquery <<'EOF'
INSERT INTO events VALUES (400, 'script_run', '2026-03-31 15:00:00');
INSERT INTO events VALUES (401, 'script_end', '2026-03-31 15:01:00');
EOF
```

## INSERT with SELECT

You can populate a table from a query result in a single statement - covered in depth in the INSERT SELECT post - but the basic form is:

```sql
INSERT INTO events
SELECT
    id,
    action,
    ts
FROM staging_events
WHERE ts >= '2026-03-31 00:00:00';
```

## Best Practices

- Keep insert batches large (aim for 1 MB-1 GB of uncompressed data per INSERT).
- Avoid inserting into a table from many concurrent threads without batching; use async inserts or a buffer table instead.
- Use `INSERT INTO ... FORMAT` with binary or columnar formats (Native, Parquet) for the highest throughput.
- Monitor `system.parts` to verify that part counts stay manageable.

```sql
-- Check active parts per table
SELECT
    table,
    count()        AS parts,
    sum(rows)      AS total_rows,
    formatReadableSize(sum(bytes_on_disk)) AS disk_size
FROM system.parts
WHERE active AND database = currentDatabase()
GROUP BY table
ORDER BY parts DESC;
```

## Summary

ClickHouse INSERT is straightforward but rewards batching. Use `INSERT INTO ... VALUES` for small ad-hoc writes, the HTTP interface or `clickhouse-client` for scripted pipelines, and always prefer large batches over many small inserts. Keeping part counts low is the key to sustained write performance.
