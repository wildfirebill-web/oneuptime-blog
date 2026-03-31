# How to Use lz4Compress() and lz4HCCompress() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Compression, LZ4, String Function, Performance

Description: Learn how lz4Compress() and lz4HCCompress() apply LZ4 and LZ4 High Compression to String values in ClickHouse, providing the fastest available in-query compression for throughput-sensitive workloads.

---

`lz4Compress(data)` compresses a `String` using the LZ4 algorithm, which prioritizes extreme compression and decompression speed over compression ratio. `lz4HCCompress(data [, level])` uses LZ4 High Compression mode, which sacrifices some speed during compression to achieve better compression ratios while keeping decompression as fast as standard LZ4. `lz4Decompress(data)` decompresses data produced by either function. LZ4 is the default codec used by ClickHouse's native network protocol and one of the most common storage codecs.

## Basic lz4Compress() Usage

```sql
-- Compress a repetitive string with LZ4
SELECT
    length(original)                AS original_bytes,
    length(lz4Compress(original))   AS compressed_bytes,
    round(
        length(lz4Compress(original)) * 100.0 / length(original),
        1
    )                               AS ratio_pct
FROM (
    SELECT repeat('{"event":"page_view","user_id":12345,"page":"/home"}', 100) AS original
);
```

```text
original_bytes  compressed_bytes  ratio_pct
5100            90                1.8
```

## lz4HCCompress() - High Compression Mode

```sql
SELECT
    length(original)                    AS original_bytes,
    length(lz4Compress(original))       AS lz4_bytes,
    length(lz4HCCompress(original))     AS lz4hc_bytes,
    round(length(lz4Compress(original))   * 100.0 / length(original), 1) AS lz4_ratio_pct,
    round(length(lz4HCCompress(original)) * 100.0 / length(original), 1) AS lz4hc_ratio_pct
FROM (
    SELECT repeat('{"event":"page_view","user_id":12345,"page":"/home"}', 100) AS original
);
```

```text
original_bytes  lz4_bytes  lz4hc_bytes  lz4_ratio_pct  lz4hc_ratio_pct
5100            90         71           1.8            1.4
```

LZ4 HC achieves a better ratio; both decompress at the same speed.

## lz4HCCompress Levels

`lz4HCCompress` accepts a level from 1 (faster, less compression) to 12 (slowest, best compression).

```sql
SELECT
    level,
    length(lz4HCCompress(payload, level))   AS compressed_bytes,
    round(
        length(lz4HCCompress(payload, level)) * 100.0 / length(payload),
        2
    )                                       AS ratio_pct
FROM (
    SELECT
        number AS level,
        repeat('{"user":12345,"action":"click","page":"/product/99"}', 300) AS payload
    FROM numbers(1, 13)
)
ORDER BY level;
```

## Round-Trip Decompression

Both `lz4Compress` and `lz4HCCompress` produce data decompressible with `lz4Decompress`.

```sql
-- Standard LZ4 round-trip
SELECT
    original,
    lz4Decompress(lz4Compress(original)) AS from_lz4
FROM (SELECT 'Fast LZ4 compression in ClickHouse.' AS original);
```

```text
original                                from_lz4
Fast LZ4 compression in ClickHouse.    Fast LZ4 compression in ClickHouse.
```

```sql
-- LZ4 HC round-trip
SELECT
    original,
    lz4Decompress(lz4HCCompress(original, 9)) AS from_lz4hc
FROM (SELECT 'High Compression LZ4 round-trip test.' AS original);
```

```text
original                                from_lz4hc
High Compression LZ4 round-trip test.  High Compression LZ4 round-trip test.
```

## Storing LZ4-Compressed Blobs

```sql
CREATE TABLE lz4_event_store
(
    event_id        UInt64,
    event_time      DateTime,
    compressed_body String
)
ENGINE = MergeTree()
ORDER BY (event_time, event_id);
```

```sql
INSERT INTO lz4_event_store (event_id, event_time, compressed_body)
SELECT
    number,
    now() - (number * 3),
    lz4Compress(
        concat(
            '{"event_id":', toString(number),
            ',"type":"click","user_id":', toString(rand() % 100000),
            ',"url":"/page/', toString(rand() % 500), '"}'
        )
    )
FROM numbers(1, 10001);
```

```sql
-- Read back with decompression
SELECT
    event_id,
    event_time,
    lz4Decompress(compressed_body) AS event_body
FROM lz4_event_store
WHERE event_id <= 5
ORDER BY event_id;
```

## Speed Comparison: LZ4 vs zstd vs gzip

```sql
-- Relative throughput note: LZ4 typically compresses at 500+ MB/s vs ~200 MB/s for zstd
-- Measure compressed sizes on a sample payload
SELECT
    algorithm,
    length(compressed)  AS compressed_bytes,
    round(length(compressed) * 100.0 / length(payload), 1) AS ratio_pct
FROM (
    SELECT payload,
           'lz4'         AS algorithm, lz4Compress(payload)      AS compressed
    FROM (SELECT repeat('{"event":"view","id":42}', 500) AS payload)

    UNION ALL

    SELECT payload,
           'lz4hc(9)'    AS algorithm, lz4HCCompress(payload, 9)  AS compressed
    FROM (SELECT repeat('{"event":"view","id":42}', 500) AS payload)

    UNION ALL

    SELECT payload,
           'zstd(3)'     AS algorithm, zstdCompress(payload, 3)   AS compressed
    FROM (SELECT repeat('{"event":"view","id":42}', 500) AS payload)

    UNION ALL

    SELECT payload,
           'gzip(6)'     AS algorithm, gzipCompress(payload, 6)   AS compressed
    FROM (SELECT repeat('{"event":"view","id":42}', 500) AS payload)
)
ORDER BY ratio_pct;
```

## Using LZ4 as a Column Codec

For storage-level compression, use ClickHouse codecs directly rather than per-value functions.

```sql
-- LZ4 codec applied at storage layer (recommended for columnar compression)
CREATE TABLE events_lz4_codec
(
    event_id   UInt64 CODEC(LZ4),
    user_id    UInt64 CODEC(LZ4),
    payload    String CODEC(LZ4HC(9)),
    event_time DateTime CODEC(DoubleDelta, LZ4)
)
ENGINE = MergeTree()
ORDER BY event_time;
```

Storage-layer codecs compress entire data blocks together, achieving significantly better ratios than per-row function calls.

## Estimating Bulk Storage Savings

```sql
SELECT
    count()                                                  AS rows,
    sum(length(lz4Decompress(compressed_body)))              AS total_original_bytes,
    sum(length(compressed_body))                             AS total_compressed_bytes,
    round(
        (1 - sum(length(compressed_body)) * 1.0
               / sum(length(lz4Decompress(compressed_body)))) * 100,
        1
    )                                                        AS space_savings_pct
FROM lz4_event_store;
```

## Summary

`lz4Compress(data)` provides the fastest possible in-query compression with moderate ratio, `lz4HCCompress(data [, level])` trades slower compression for a better ratio while maintaining the same fast decompression speed, and `lz4Decompress(data)` decompresses output from either. Use LZ4 when throughput is the priority - such as buffering high-frequency event streams into compressed blobs. Use LZ4 HC when you want better space savings with no decompression penalty. For column-level storage compression at scale, prefer ClickHouse's `CODEC(LZ4)` or `CODEC(LZ4HC(level))` storage codecs over per-value function calls.
