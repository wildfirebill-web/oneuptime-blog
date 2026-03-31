# How to Handle Large String Columns in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, String Column, Compression, Storage, Performance, ZSTD

Description: Techniques for storing and querying large string columns in ClickHouse efficiently, including codec selection, tokenization, and full-text search alternatives.

---

## The Cost of Large Strings

ClickHouse stores strings as variable-length byte arrays. Large strings (log messages, JSON blobs, URLs, HTML) inflate storage, slow down compression, and increase memory use during queries. Understanding your options is key to keeping performance acceptable.

## Codec Selection for Large Strings

The default LZ4 codec is fast but may not compress large strings well. ZSTD gives better compression ratios:

```sql
CREATE TABLE app_logs (
    ts      DateTime,
    host    LowCardinality(String),
    message String CODEC(ZSTD(3))
) ENGINE = MergeTree()
ORDER BY (host, ts);
```

For very repetitive text (structured log lines), ZSTD(6) or higher may reduce storage by 5-10x vs raw.

## Using LowCardinality for Repeated Strings

If a string column has limited unique values (status codes, log levels, service names), use `LowCardinality`:

```sql
log_level  LowCardinality(String),
service    LowCardinality(String),
```

`LowCardinality` stores a dictionary of unique values and replaces string data with integer keys. This can reduce storage 10x and speed up GROUP BY and WHERE queries dramatically.

## Truncating and Extracting Before Storage

If you only query parts of a large string, extract what you need at ingest:

```sql
CREATE MATERIALIZED VIEW logs_parsed
ENGINE = MergeTree() ORDER BY (host, ts) AS
SELECT
    ts,
    host,
    substring(message, 1, 200)  AS short_msg,
    extract(message, 'error_code=([0-9]+)') AS error_code
FROM raw_logs;
```

Store the full message in cold/archive storage and the parsed fields in a hot table.

## Full-Text Search Limitations

ClickHouse is not a full-text search engine. For substring search, use `LIKE` or `positionUTF8`:

```sql
SELECT ts, message
FROM app_logs
WHERE message LIKE '%connection refused%'
  AND ts >= now() - INTERVAL 1 HOUR;
```

For faster substring search, enable a token bloom filter:

```sql
ALTER TABLE app_logs
ADD INDEX msg_idx message TYPE tokenbf_v1(32768, 3, 0) GRANULARITY 1;
```

This index pre-filters data granules before scanning, reducing IO for LIKE queries.

## Avoiding Large String in ORDER BY

Never put large string columns in the ORDER BY key - sorting on variable-length data increases merge time significantly:

```sql
-- Bad
ORDER BY (message, ts)

-- Good
ORDER BY (host, ts)
```

## Compressing JSON Blobs

If storing JSON, consider extracting hot fields via materialized view and compressing the raw JSON blob separately:

```sql
raw_json String CODEC(ZSTD(6))
```

## Summary

Handle large string columns in ClickHouse by applying ZSTD codecs, using LowCardinality for low-cardinality fields, extracting relevant substrings at ingest time, and adding token bloom filter indexes for substring searches. Avoid including large strings in sort keys, and keep raw blobs out of hot query paths.
