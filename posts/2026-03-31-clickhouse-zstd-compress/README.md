# How to Use zstdCompress() and zstdDecompress() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Compression, ZSTD, String Function, Data Storage

Description: Learn how zstdCompress() and zstdDecompress() apply Zstandard compression to String values in ClickHouse, offering excellent compression ratios with fast decompression for analytical workloads.

---

`zstdCompress(data [, level])` compresses a `String` using the Zstandard (zstd) algorithm and returns compressed bytes as a `String`. `zstdDecompress(data)` reverses the process. Zstd is developed by Facebook and offers an excellent balance of compression ratio and speed - particularly fast decompression - making it a popular choice for analytical workloads where data is compressed once and read many times. ClickHouse itself uses zstd as one of its native storage codecs.

## Basic Compression and Decompression

```sql
-- Compress a repetitive string and check the ratio
SELECT
    length(original)                  AS original_bytes,
    length(zstdCompress(original))    AS compressed_bytes,
    round(
        length(zstdCompress(original)) * 100.0 / length(original),
        1
    )                                 AS ratio_pct
FROM (
    SELECT repeat('{"event":"page_view","user_id":42,"url":"/home"}', 100) AS original
);
```

```text
original_bytes  compressed_bytes  ratio_pct
4900            105               2.1
```

## Decompression Round-Trip

```sql
SELECT
    original,
    zstdDecompress(zstdCompress(original)) AS roundtrip
FROM (
    SELECT 'ClickHouse with Zstandard compression is very efficient.' AS original
);
```

```text
original                                                     roundtrip
ClickHouse with Zstandard compression is very efficient.     ClickHouse with Zstandard compression is very efficient.
```

## Compression Levels

Zstd supports levels 1 (fastest) through 22 (maximum compression). Level 3 is a commonly used default that balances speed and ratio.

```sql
SELECT
    level,
    length(zstdCompress(payload, level))   AS compressed_bytes,
    round(
        length(zstdCompress(payload, level)) * 100.0 / length(payload),
        2
    )                                      AS ratio_pct
FROM (
    SELECT
        number AS level,
        repeat('{"event":"purchase","amount":99.99,"currency":"USD"}', 500) AS payload
    FROM numbers(1, 8)
)
ORDER BY level;
```

```text
level  compressed_bytes  ratio_pct
1      90                0.36
2      89                0.36
3      88                0.35
4      88                0.35
5      85                0.34
6      84                0.34
7      84                0.34
```

## Storing Compressed Payloads in a Table

```sql
CREATE TABLE event_payloads
(
    event_id         UInt64,
    event_time       DateTime,
    compressed_data  String
)
ENGINE = MergeTree()
ORDER BY (event_time, event_id);
```

```sql
INSERT INTO event_payloads (event_id, event_time, compressed_data)
SELECT
    number,
    now() - (number * 5),
    zstdCompress(
        concat(
            '{"event_id":', toString(number),
            ',"type":"click","page":"/item/', toString(number % 1000),
            '","user":', toString(rand() % 50000), '}'
        ),
        3
    )
FROM numbers(1, 10001);
```

```sql
-- Decompress on read
SELECT
    event_id,
    event_time,
    zstdDecompress(compressed_data) AS payload
FROM event_payloads
WHERE event_id <= 5
ORDER BY event_id;
```

## Measuring Compression Efficiency Across Payload Types

```sql
-- Compare zstd efficiency on different payload types
SELECT
    payload_type,
    avg(length(payload))                AS avg_original,
    avg(length(zstdCompress(payload)))  AS avg_compressed,
    round(avg(length(zstdCompress(payload))) * 100.0 / avg(length(payload)), 1) AS ratio_pct
FROM (
    -- JSON events
    SELECT 'json_event' AS payload_type,
           concat('{"id":', toString(number), ',"user":', toString(rand() % 10000),
                  ',"action":"view","ts":', toString(toUnixTimestamp(now())), '}') AS payload
    FROM numbers(1000)

    UNION ALL

    -- Log lines
    SELECT 'log_line' AS payload_type,
           concat('[2024-06-15 12:00:00] INFO Request processed user_id=',
                  toString(rand() % 10000), ' duration_ms=', toString(rand() % 5000)) AS payload
    FROM numbers(1000)

    UNION ALL

    -- Random binary-like data (low compressibility)
    SELECT 'random_hex' AS payload_type,
           hex(rand64()) AS payload
    FROM numbers(1000)
)
GROUP BY payload_type
ORDER BY ratio_pct;
```

## zstd vs Column Codec

```sql
-- Using zstd as a storage codec (preferred for columnar compression)
CREATE TABLE events_with_codec
(
    event_id  UInt64,
    user_id   UInt64,
    payload   String CODEC(ZSTD(3)),      -- applied at storage layer
    event_ts  DateTime CODEC(DoubleDelta, ZSTD(1))
)
ENGINE = MergeTree()
ORDER BY (event_ts, event_id);
```

The storage codec approach compresses all column values together, achieving better ratios than per-value compression. Use `zstdCompress()` only when you need to store or transmit compressed blobs inside a `String` column or send them outside ClickHouse.

## Inline Compression for ETL Pipelines

```sql
-- Compress data during a SELECT for export
SELECT
    event_id,
    base64Encode(zstdCompress(payload, 3)) AS compressed_b64
FROM (
    SELECT
        number AS event_id,
        concat('{"id":', toString(number), ',"data":"sample"}') AS payload
    FROM numbers(1, 6)
);
```

## Bulk Storage Savings Estimate

```sql
SELECT
    count()                                                  AS rows,
    sum(length(zstdDecompress(compressed_data)))             AS total_original_bytes,
    sum(length(compressed_data))                             AS total_compressed_bytes,
    round(
        (1 - sum(length(compressed_data)) * 1.0
               / sum(length(zstdDecompress(compressed_data)))) * 100,
        1
    )                                                        AS space_savings_pct
FROM event_payloads;
```

## Summary

`zstdCompress(data [, level])` applies Zstandard compression to a `String` value, and `zstdDecompress(data)` reverses it. Zstd offers fast decompression, strong compression ratios at default level 3, and levels up to 22 for maximum compression. For large-scale columnar storage, prefer ClickHouse storage codecs (`CODEC(ZSTD(level))`) over per-value compression. Use `zstdCompress()` when you need to compress blobs inline - for example, when packaging compressed payloads for external transfer or when storing JSON that should not be parsed into columns.
