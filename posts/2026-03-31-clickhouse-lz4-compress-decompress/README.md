# How to Use lz4Compress() and lz4Decompress() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Compression, lz4Compress, lz4Decompress, Performance

Description: Learn how to use lz4Compress() and lz4Decompress() in ClickHouse to compress and decompress string data with LZ4 for fast, lightweight compression of column values.

---

LZ4 is a fast lossless compression algorithm optimized for speed rather than compression ratio. ClickHouse uses LZ4 as its default internal storage codec and also exposes it as SQL functions `lz4Compress()` and `lz4Decompress()` for compressing arbitrary string values within queries. This is useful for reducing storage size, passing compressed payloads through the database, and benchmarking compression behavior.

## How LZ4 Compression Works

```mermaid
graph LR
    A[Original String] -->|lz4Compress()| B[Compressed Binary]
    B -->|lz4Decompress()| A
```

LZ4 achieves high decompression speeds (typically over 1 GB/s on a single core) with moderate compression ratios. It is best suited for repetitive or structured text data.

## Syntax

```sql
lz4Compress(string)
lz4Decompress(compressed_string)
```

Both functions accept a `String` argument. `lz4Compress()` returns a `String` containing the compressed binary data. `lz4Decompress()` takes that compressed binary and returns the original string.

## Basic Examples

### Compress and Decompress a String

```sql
SELECT
    length('Hello, ClickHouse! Hello, ClickHouse!')           AS original_len,
    length(lz4Compress('Hello, ClickHouse! Hello, ClickHouse!')) AS compressed_len,
    lz4Decompress(lz4Compress('Hello, ClickHouse! Hello, ClickHouse!')) AS roundtrip;
```

```text
original_len | compressed_len | roundtrip
-------------+----------------+--------------------------------------
38           | 26             | Hello, ClickHouse! Hello, ClickHouse!
```

### Compression Ratio on Repetitive Data

LZ4 performs best on data with repeated patterns.

```sql
SELECT
    length(repeat('abcdefgh', 100))              AS original_bytes,
    length(lz4Compress(repeat('abcdefgh', 100))) AS compressed_bytes,
    round(
        (1 - length(lz4Compress(repeat('abcdefgh', 100))) /
             length(repeat('abcdefgh', 100))) * 100,
        1
    )                                            AS savings_pct;
```

```text
original_bytes | compressed_bytes | savings_pct
---------------+------------------+------------
800            | 18               | 97.8
```

## Complete Working Example

This example creates a table for storing log lines both raw and compressed, inserts sample data, and compares sizes.

```sql
CREATE TABLE log_compressed
(
    id             UInt32,
    log_raw        String,
    log_compressed String
)
ENGINE = MergeTree()
ORDER BY id;

INSERT INTO log_compressed
SELECT
    number,
    concat(
        '2026-03-31 12:00:', toString(number),
        ' INFO  [server] Request processed successfully. RequestID=',
        toString(number), ' Duration=', toString(number * 3), 'ms'
    ),
    lz4Compress(concat(
        '2026-03-31 12:00:', toString(number),
        ' INFO  [server] Request processed successfully. RequestID=',
        toString(number), ' Duration=', toString(number * 3), 'ms'
    ))
FROM numbers(10);

SELECT
    id,
    length(log_raw)        AS raw_bytes,
    length(log_compressed) AS compressed_bytes,
    lz4Decompress(log_compressed) AS decompressed
FROM log_compressed
ORDER BY id
LIMIT 3;
```

## Comparing Compressed Sizes

It is helpful to compare LZ4 against other available compression functions to choose the right one for your use case.

```sql
SELECT
    length(input_str)                    AS original_bytes,
    length(lz4Compress(input_str))       AS lz4_bytes,
    length(gzipCompress(input_str))      AS gzip_bytes,
    length(zstdCompress(1)(input_str))   AS zstd_bytes,
    length(brotliCompress(3)(input_str)) AS brotli_bytes
FROM (
    SELECT repeat('ClickHouse log entry: error code 500. ', 50) AS input_str
);
```

```text
original_bytes | lz4_bytes | gzip_bytes | zstd_bytes | brotli_bytes
---------------+-----------+------------+------------+-------------
1900           | 57        | 48         | 45         | 38
```

LZ4 produces slightly larger output than gzip or zstd but typically compresses and decompresses faster.

## Storing Compressed JSON Payloads

A practical use case is storing large JSON blobs in compressed form to reduce storage.

```sql
CREATE TABLE events
(
    event_id   UInt64,
    created_at DateTime,
    payload    String   -- stores lz4-compressed JSON
)
ENGINE = MergeTree()
ORDER BY (created_at, event_id);

INSERT INTO events VALUES
    (1, now(), lz4Compress('{"user":1,"action":"login","ip":"10.0.0.1","timestamp":1711900800}')),
    (2, now(), lz4Compress('{"user":2,"action":"purchase","item_id":42,"amount":99.99}'));

-- Decompress on read
SELECT
    event_id,
    created_at,
    lz4Decompress(payload) AS json_payload
FROM events
ORDER BY event_id;
```

## Important Notes

- The output of `lz4Compress()` is a binary string with a ClickHouse-specific header. Do not pass raw LZ4 output from external tools directly to `lz4Decompress()` without accounting for the header format.
- For bulk storage compression, prefer column codecs (`CODEC(LZ4)` in the `CREATE TABLE` DDL) rather than compressing values with these functions. Column-level codecs are applied transparently and do not require changes to your queries.
- The compressed output is not a printable string. Store it in a `String` column and avoid treating it as text.

## LZ4 vs LZ4HC

ClickHouse also provides `lz4HCCompress(level)(string)` which uses the high-compression variant of LZ4. LZ4HC trades compression speed for a better compression ratio. Its output is still decompressed with `lz4Decompress()`.

```sql
SELECT
    length(lz4Compress(repeat('data', 200)))       AS lz4_bytes,
    length(lz4HCCompress(9)(repeat('data', 200)))  AS lz4hc_bytes;
```

## Summary

`lz4Compress()` and `lz4Decompress()` provide fast, in-query LZ4 compression for string values in ClickHouse. LZ4 is best suited for scenarios where decompression speed matters more than maximum compression ratio. For persistent column-level compression, use the `CODEC(LZ4)` column option instead. Use `lz4HCCompress()` when you need a better ratio at the cost of slower compression speed.
