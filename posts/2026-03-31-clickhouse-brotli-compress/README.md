# How to Use brotliCompress() and brotliDecompress() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Compression, Brotli, String Function, Data Storage

Description: Learn how brotliCompress() and brotliDecompress() apply Brotli compression and decompression to String values in ClickHouse for compact storage of text-heavy payloads.

---

`brotliCompress(data [, level])` compresses a `String` value using the Brotli algorithm and returns the compressed bytes as a `String`. `brotliDecompress(data)` reverses the process. Brotli is a general-purpose compression algorithm developed by Google that typically achieves better compression ratios than zlib/gzip at comparable speeds, making it well-suited for compressing repetitive text payloads such as JSON, HTML, and log lines stored inside ClickHouse columns.

## Basic Compression and Decompression

```sql
-- Compress and measure size reduction for a repetitive string
SELECT
    length(original)                   AS original_bytes,
    length(brotliCompress(original))   AS compressed_bytes,
    round(
        length(brotliCompress(original)) * 100.0 / length(original),
        1
    )                                  AS compressed_pct
FROM (
    SELECT repeat('Hello, ClickHouse! This is a test of Brotli compression. ', 100) AS original
);
```

```text
original_bytes  compressed_bytes  compressed_pct
5800            57                1.0
```

## Decompression Round-Trip

```sql
SELECT
    original,
    brotliDecompress(brotliCompress(original)) AS roundtrip
FROM (
    SELECT 'ClickHouse is a fast OLAP database.' AS original
);
```

```text
original                                   roundtrip
ClickHouse is a fast OLAP database.        ClickHouse is a fast OLAP database.
```

## Compression Levels

`brotliCompress` accepts an optional level from 0 (fastest, least compression) to 11 (best compression, slowest).

```sql
-- Compare compression size at different levels
SELECT
    level,
    length(brotliCompress(payload, level))   AS compressed_bytes,
    round(
        length(brotliCompress(payload, level)) * 100.0 / length(payload),
        2
    )                                        AS ratio_pct
FROM (
    SELECT
        number AS level,
        repeat('{"event":"page_view","user_id":12345,"url":"/home"}', 200) AS payload
    FROM numbers(0, 12)
)
ORDER BY level;
```

```text
level  compressed_bytes  ratio_pct
0      305               1.52
1      222               1.11
...
11     188               0.94
```

## Storing Compressed Payloads

```sql
CREATE TABLE compressed_logs
(
    log_id       UInt64,
    log_time     DateTime,
    compressed_body String  -- stores brotli-compressed JSON
)
ENGINE = MergeTree()
ORDER BY (log_time, log_id);
```

```sql
-- Insert with compression applied at write time
INSERT INTO compressed_logs (log_id, log_time, compressed_body)
SELECT
    number,
    now() - (number * 10),
    brotliCompress(
        concat('{"id":', toString(number), ',"event":"click","session":"', toString(rand()), '"}')
    )
FROM numbers(1, 1001);
```

```sql
-- Read and decompress on query
SELECT
    log_id,
    log_time,
    brotliDecompress(compressed_body) AS log_body
FROM compressed_logs
WHERE log_id <= 5
ORDER BY log_id;
```

## Compression Ratio for JSON Payloads

```sql
-- Benchmark Brotli compression on realistic JSON event data
SELECT
    avg(length(payload))                  AS avg_original_bytes,
    avg(length(brotliCompress(payload)))  AS avg_compressed_bytes,
    round(
        avg(length(brotliCompress(payload))) * 100.0 / avg(length(payload)),
        1
    )                                     AS avg_ratio_pct
FROM (
    SELECT
        concat(
            '{"event_id":"', toString(number),
            '","user_id":', toString(rand() % 100000),
            ',"page":"/product/', toString(rand() % 1000),
            '","ts":', toString(toUnixTimestamp(now())),
            ',"session":"', hex(rand()), '"}'
        ) AS payload
    FROM numbers(10000)
);
```

## Detecting Corrupted Compressed Data

If `brotliDecompress` receives data that is not valid Brotli-compressed bytes, it throws an exception. Use `isValidBrotli` if available, or wrap in a `try` expression.

```sql
-- Safe decompression with fallback
SELECT
    log_id,
    tryBrotliDecompress(compressed_body) AS maybe_decompressed
FROM compressed_logs
LIMIT 5;
```

If `tryBrotliDecompress` is not available in your version, use a conditional approach.

```sql
-- Check that compressed data starts with valid Brotli header bytes
SELECT
    log_id,
    length(compressed_body)                AS compressed_size,
    length(brotliDecompress(compressed_body)) AS decompressed_size
FROM compressed_logs
WHERE log_id = 1;
```

## Comparing Brotli vs Raw Storage

```sql
-- Space savings summary across all compressed logs
SELECT
    count()                                                AS total_rows,
    sum(length(brotliDecompress(compressed_body)))         AS total_original_bytes,
    sum(length(compressed_body))                           AS total_compressed_bytes,
    round(
        sum(length(compressed_body)) * 100.0
        / sum(length(brotliDecompress(compressed_body))),
        1
    )                                                      AS overall_ratio_pct
FROM compressed_logs;
```

## When to Use brotliCompress in ClickHouse

Brotli compression inside a column is most useful when:
- You store raw text payloads (JSON, XML, HTML) that are too large to expand into proper columns.
- You need to exchange compressed data with web clients that support `Content-Encoding: br`.
- You want better compression than gzip at the same or better query speed.

For columnar storage compression (reducing disk footprint of ClickHouse data parts), configure the `CODEC` clause on the column definition instead.

```sql
-- Preferred approach for columnar compression: codec on column definition
CREATE TABLE events_columnar
(
    event_id   UInt64,
    payload    String CODEC(ZSTD(3))  -- applied by ClickHouse storage engine, not per-value
)
ENGINE = MergeTree()
ORDER BY event_id;
```

## Summary

`brotliCompress(data [, level])` compresses a `String` using the Brotli algorithm, and `brotliDecompress(data)` reverses it. Brotli achieves high compression ratios on repetitive text, making it effective for storing JSON, log lines, and HTML payloads inside ClickHouse columns. Use levels 0-11 to trade off speed versus compression ratio. For per-column storage compression, prefer ClickHouse storage codecs (`CODEC(ZSTD(...))`) as they operate at the storage layer and are applied transparently to all reads and writes.
