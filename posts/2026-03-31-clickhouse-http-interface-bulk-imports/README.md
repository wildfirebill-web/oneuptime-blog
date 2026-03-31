# How to Use ClickHouse HTTP Interface for Bulk Imports

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, HTTP Interface, Bulk Import, CSV, JSONEachRow

Description: Learn how to use the ClickHouse HTTP interface to bulk import data from CSV, JSON, and Parquet files efficiently using curl and custom scripts.

---

ClickHouse's HTTP interface accepts bulk data imports directly over HTTP, making it easy to load files from scripts, CI pipelines, and ETL jobs without a dedicated client.

## Basic JSON Bulk Insert

Send multiple rows as newline-delimited JSON:

```bash
curl -X POST \
  'http://clickhouse:8123/?query=INSERT+INTO+events+FORMAT+JSONEachRow' \
  -H 'X-ClickHouse-User: default' \
  -H 'X-ClickHouse-Key: secret' \
  --data-binary @events.ndjson
```

Where `events.ndjson` contains one JSON object per line.

## Bulk Insert from CSV

```bash
curl -X POST \
  'http://clickhouse:8123/?query=INSERT+INTO+events+FORMAT+CSVWithNames' \
  -H 'Content-Type: text/csv' \
  -H 'X-ClickHouse-User: default' \
  -H 'X-ClickHouse-Key: secret' \
  --data-binary @events.csv
```

ClickHouse infers column mapping from the CSV header row with `CSVWithNames`.

## Compressed Bulk Insert

Reduce transfer time with gzip compression:

```bash
gzip -c events.ndjson | curl -X POST \
  'http://clickhouse:8123/?query=INSERT+INTO+events+FORMAT+JSONEachRow' \
  -H 'Content-Encoding: gzip' \
  -H 'X-ClickHouse-User: default' \
  -H 'X-ClickHouse-Key: secret' \
  --data-binary @-
```

## Streaming Large Files

For files larger than memory, stream directly without loading the whole file:

```bash
cat large_events.ndjson | curl -X POST \
  'http://clickhouse:8123/?query=INSERT+INTO+events+FORMAT+JSONEachRow&async_insert=1' \
  -H 'Transfer-Encoding: chunked' \
  -H 'X-ClickHouse-User: default' \
  --data-binary @-
```

## Parquet Import via S3 Function

For Parquet files already in S3, use the `s3` table function:

```sql
INSERT INTO events
SELECT * FROM s3(
    'https://s3.amazonaws.com/mybucket/events/*.parquet',
    'AWS_KEY', 'AWS_SECRET',
    'Parquet'
);
```

## Parallel Import with xargs

Split a large file into chunks and import in parallel:

```bash
split -l 100000 events.ndjson chunk_
ls chunk_* | xargs -P 8 -I{} curl -X POST \
  'http://clickhouse:8123/?query=INSERT+INTO+events+FORMAT+JSONEachRow' \
  -H 'X-ClickHouse-User: default' \
  --data-binary @{}
```

## Summary

ClickHouse's HTTP interface handles bulk imports from JSON, CSV, and Parquet via curl. Use gzip compression to reduce transfer time, stream large files with chunked encoding, and parallelize with `xargs` for maximum throughput. Enable `async_insert=1` to reduce part creation overhead.
