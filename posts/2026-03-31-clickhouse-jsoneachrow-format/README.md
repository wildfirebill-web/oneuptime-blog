# How to Use JSONEachRow Format in ClickHouse

Author: [oneuptime](https://github.com/oneuptime)

Tags: ClickHouse, JSON, Data Engineering, Logging

Description: Learn how to use ClickHouse's JSONEachRow format for importing NDJSON logs, exporting query results, and building real-time JSON ingestion pipelines.

## What Is JSONEachRow?

`JSONEachRow` (also called NDJSON or newline-delimited JSON) stores one JSON object per line. It is the most common format for log data, event streams, and API payloads. ClickHouse's `JSONEachRow` format is one of the most widely used formats in production because nearly every data producer can emit newline-delimited JSON.

Example input file:

```text
{"id":1,"user":"alice","action":"login","ts":"2025-01-01 10:00:00"}
{"id":2,"user":"bob","action":"purchase","ts":"2025-01-01 10:01:00"}
{"id":3,"user":"carol","action":"logout","ts":"2025-01-01 10:02:00"}
```

## Reading JSONEachRow Files

Query a file directly:

```sql
SELECT *
FROM file('events.ndjson', JSONEachRow)
LIMIT 10;
```

ClickHouse infers types from the first rows. Inspect the schema:

```sql
DESCRIBE file('events.ndjson', JSONEachRow);
```

## Inserting JSONEachRow Data

Load from a file into a table:

```sql
CREATE TABLE events
(
    id      UInt64,
    user    String,
    action  LowCardinality(String),
    ts      DateTime
)
ENGINE = MergeTree()
ORDER BY (ts, user);

INSERT INTO events
SELECT *
FROM file('events.ndjson', JSONEachRow);
```

Using `clickhouse-client` with stdin:

```bash
cat events.ndjson | clickhouse-client \
  --query "INSERT INTO events FORMAT JSONEachRow"
```

## Inserting via HTTP Interface

The HTTP interface is particularly convenient for sending JSON events from applications:

```bash
curl -X POST 'http://localhost:8123/?query=INSERT+INTO+events+FORMAT+JSONEachRow' \
  --data-binary @events.ndjson
```

Or from an application:

```bash
echo '{"id":4,"user":"dave","action":"login","ts":"2025-01-01 11:00:00"}' | \
  curl -X POST 'http://localhost:8123/?query=INSERT+INTO+events+FORMAT+JSONEachRow' \
  --data-binary @-
```

## Exporting Query Results

Export any query result as NDJSON:

```sql
SELECT id, user, action, ts
FROM events
WHERE action = 'purchase'
FORMAT JSONEachRow;
```

Into a file:

```sql
SELECT *
FROM events
INTO OUTFILE 'purchases.ndjson'
FORMAT JSONEachRow;
```

## Handling Missing Fields

By default, missing fields in the JSON cause an error. Enable defaults for missing keys:

```sql
SET input_format_defaults_for_omitted_fields = 1;
```

With this setting, any column not present in the JSON object uses the column's default value.

## Extra Fields

By default, ClickHouse ignores extra JSON fields that do not match table columns. This is controlled by:

```sql
SET input_format_skip_unknown_fields = 1; -- default for JSONEachRow
```

## Nested JSON Objects

ClickHouse can flatten nested JSON objects automatically. Given input:

```text
{"id":1,"user":{"name":"alice","age":30},"score":9.5}
```

Enable the setting:

```sql
SET input_format_json_read_objects_as_strings = 0;
SET input_format_flatten_nested = 1;
```

Or map the nested object as a `String` and parse it later:

```sql
CREATE TABLE events_raw (id UInt64, user String, score Float32)
ENGINE = MergeTree() ORDER BY id;

-- user will contain '{"name":"alice","age":30}'
INSERT INTO events_raw
SELECT id, user, score
FROM file('nested.ndjson', JSONEachRow);

-- Extract fields on read
SELECT id, JSONExtractString(user, 'name') AS name FROM events_raw;
```

## JSONEachRowWithProgress

For long-running HTTP queries, ClickHouse can interleave progress rows with data rows:

```sql
SELECT * FROM large_table FORMAT JSONEachRowWithProgress;
```

Each progress line looks like:

```text
{"progress":{"read_rows":"1000000","read_bytes":"52428800",...}}
{"id":1,"user":"alice",...}
```

## Useful Settings Summary

| Setting | Default | Description |
|---------|---------|-------------|
| `input_format_skip_unknown_fields` | 1 | Ignore extra JSON keys |
| `input_format_defaults_for_omitted_fields` | 0 | Use column defaults for missing keys |
| `output_format_json_quote_64bit_integers` | 1 | Wrap UInt64 in quotes |
| `output_format_json_escape_forward_slashes` | 1 | Escape `/` in strings |

## Performance Tips

1. **Batch your inserts** - send thousands of rows per request rather than one per request.
2. **Use async_insert mode** for high-throughput event ingestion from many small producers.
3. **Prefer binary formats** (Parquet, Arrow) for large bulk loads; JSONEachRow has parsing overhead.
4. **Use `LowCardinality(String)`** for columns with limited distinct values to reduce memory usage.

## Conclusion

JSONEachRow is the universal entry point for getting data into ClickHouse. Its simplicity makes it easy to integrate from any language or tool. For production log ingestion pipelines, combine it with ClickHouse's HTTP interface and async insert mode for high-throughput, low-latency event storage.

**Related Reading:**

- [How to Use JSONCompact Format in ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-jsoncompact-format/view)
- [How to Use Avro Format in ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-avro-format/view)
- [How to Export ClickHouse Data to Different File Formats](https://oneuptime.com/blog/post/2026-03-31-clickhouse-export-file-formats/view)
