# How to Use gzipCompress() and gzipDecompress() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Compression, Gzip, String Function, Data Storage

Description: Learn how gzipCompress() and gzipDecompress() apply gzip compression and decompression to String values in ClickHouse for web-compatible payload compression and legacy data interchange.

---

`gzipCompress(data [, level])` compresses a `String` using the gzip format and returns the compressed bytes as a `String`. `gzipDecompress(data)` decompresses gzip-formatted data back to the original bytes. Gzip is the most widely supported compression format for HTTP content encoding and file interchange, making these functions useful when ClickHouse needs to produce or consume data compatible with web servers, browsers, and tools that expect standard gzip format.

## Basic Compression and Decompression

```sql
-- Compress a string and measure the ratio
SELECT
    length(original)                  AS original_bytes,
    length(gzipCompress(original))    AS compressed_bytes,
    round(
        length(gzipCompress(original)) * 100.0 / length(original),
        1
    )                                 AS ratio_pct
FROM (
    SELECT repeat('ClickHouse gzip compression example payload. ', 100) AS original
);
```

```text
original_bytes  compressed_bytes  ratio_pct
4400            59                1.3
```

## Round-Trip Verification

```sql
SELECT
    original,
    gzipDecompress(gzipCompress(original)) AS roundtrip
FROM (
    SELECT 'The quick brown fox jumps over the lazy dog.' AS original
);
```

```text
original                                       roundtrip
The quick brown fox jumps over the lazy dog.   The quick brown fox jumps over the lazy dog.
```

## Compression Levels

`gzipCompress` accepts levels 1 (fastest) through 9 (best compression).

```sql
SELECT
    level,
    length(gzipCompress(payload, level))   AS compressed_bytes,
    round(
        length(gzipCompress(payload, level)) * 100.0 / length(payload),
        2
    )                                      AS ratio_pct
FROM (
    SELECT
        number AS level,
        repeat('{"user_id":12345,"event":"click","page":"/home","ts":1718400000}', 200) AS payload
    FROM numbers(1, 10)
)
ORDER BY level;
```

```text
level  compressed_bytes  ratio_pct
1      113               0.89
2      113               0.89
3      111               0.87
4      108               0.85
5      107               0.84
6      107               0.84
7      107               0.84
8      107               0.84
9      107               0.84
```

## Storing gzip-Compressed Payloads

```sql
CREATE TABLE gzip_payloads
(
    id              UInt64,
    created_at      DateTime,
    compressed_json String
)
ENGINE = MergeTree()
ORDER BY (created_at, id);
```

```sql
INSERT INTO gzip_payloads (id, created_at, compressed_json)
SELECT
    number,
    now() - (number * 60),
    gzipCompress(
        concat(
            '{"id":', toString(number),
            ',"event":"view","product_id":', toString(rand() % 10000),
            ',"user_id":', toString(rand() % 100000), '}'
        ),
        6
    )
FROM numbers(1, 1001);
```

```sql
-- Decompress on query
SELECT
    id,
    created_at,
    gzipDecompress(compressed_json) AS json_body
FROM gzip_payloads
WHERE id <= 5
ORDER BY id;
```

## Decompressing Gzipped Data Ingested from External Sources

A common use case is receiving gzip-compressed payloads from HTTP clients or S3 objects.

```sql
-- Decompress a gzip payload that arrived in a String column
SELECT
    row_id,
    gzipDecompress(raw_payload) AS decompressed_content
FROM incoming_raw_data
WHERE content_encoding = 'gzip'
LIMIT 10;
```

## Comparing gzip, zstd, and brotli

```sql
SELECT
    algorithm,
    length(compressed)  AS compressed_bytes,
    round(length(compressed) * 100.0 / length(payload), 1) AS ratio_pct
FROM (
    SELECT
        payload,
        'gzip(6)'    AS algorithm,
        gzipCompress(payload, 6)    AS compressed
    FROM (SELECT repeat('{"event":"page_view","uid":999,"url":"/home"}', 200) AS payload)

    UNION ALL

    SELECT
        payload,
        'zstd(3)'    AS algorithm,
        zstdCompress(payload, 3)    AS compressed
    FROM (SELECT repeat('{"event":"page_view","uid":999,"url":"/home"}', 200) AS payload)

    UNION ALL

    SELECT
        payload,
        'brotli(6)'  AS algorithm,
        brotliCompress(payload, 6)  AS compressed
    FROM (SELECT repeat('{"event":"page_view","uid":999,"url":"/home"}', 200) AS payload)
)
ORDER BY ratio_pct;
```

## Producing gzip Output for HTTP Endpoints

```sql
-- Compress API response payloads for Content-Encoding: gzip delivery
SELECT
    base64Encode(gzipCompress(response_body, 6)) AS compressed_b64
FROM (
    SELECT
        concat(
            '{"users":[',
            arrayStringConcat(
                arrayMap(i -> concat('{"id":', toString(i), ',"name":"user', toString(i), '"}'),
                         range(1, 11)),
                ','
            ),
            ']}'
        ) AS response_body
);
```

## Bulk Storage Savings

```sql
SELECT
    count()                                                    AS rows,
    sum(length(gzipDecompress(compressed_json)))               AS total_original_bytes,
    sum(length(compressed_json))                               AS total_compressed_bytes,
    round(
        (1 - sum(length(compressed_json)) * 1.0
               / sum(length(gzipDecompress(compressed_json)))) * 100,
        1
    )                                                          AS space_savings_pct
FROM gzip_payloads;
```

## Handling NULL and Empty Inputs

```sql
SELECT
    gzipCompress('')     AS empty_compressed,
    length(gzipCompress('')) AS empty_compressed_len;
-- gzip adds a 20-byte header+footer even for empty input
```

```text
empty_compressed_len
20
```

## Summary

`gzipCompress(data [, level])` produces standard gzip-format compressed bytes from a `String`, and `gzipDecompress(data)` decompresses them. Use levels 1-9 to trade speed for compression ratio, with level 6 being a common default. Gzip is the right choice when interoperating with systems that expect standard gzip format, such as web servers, browsers, S3 objects with gzip encoding, or legacy ETL tools. For purely internal ClickHouse storage compression, zstd (`CODEC(ZSTD(...))`) or lz4 column codecs are faster and more efficient.
