# How to Use LZ4 Compression Codec in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Compression, LZ4, Database, Performance, Storage

Description: Learn how to use the LZ4 compression codec in ClickHouse for fast reads and writes with minimal CPU overhead on analytical tables.

---

LZ4 is the default compression codec in ClickHouse. It was designed from the ground up to maximize decompression throughput, often exceeding memory bandwidth limits. For workloads that prioritize query speed over storage savings, LZ4 is the right choice.

## Why LZ4

LZ4 trades compression ratio for raw speed. It compresses and decompresses data faster than any other general-purpose codec available in ClickHouse. This matters because ClickHouse queries are frequently I/O-bound when scanning large column files. Faster decompression means less time spent in the CPU between reading bytes off disk and processing them.

Key properties:

- Decompression throughput: multiple GB/s per core
- Compression ratio: moderate (typically 2-4x for mixed data)
- CPU cost: very low

## Default Behavior

When you create a MergeTree table without specifying a codec, ClickHouse uses LZ4 automatically:

```sql
CREATE TABLE page_views
(
    view_id    UInt64,
    session_id UInt64,
    url        String,
    duration   UInt32,
    ts         DateTime
)
ENGINE = MergeTree()
ORDER BY (ts, session_id);
```

All columns above implicitly use `CODEC(LZ4)`.

## Explicit LZ4 Codec

You can be explicit about LZ4 to make your schema self-documenting or to mix codecs on a single table:

```sql
CREATE TABLE page_views_explicit
(
    view_id    UInt64   CODEC(LZ4),
    session_id UInt64   CODEC(LZ4),
    url        String   CODEC(LZ4),
    duration   UInt32   CODEC(LZ4),
    ts         DateTime CODEC(LZ4)
)
ENGINE = MergeTree()
ORDER BY (ts, session_id);
```

## LZ4HC: High-Compression Variant

ClickHouse also ships `LZ4HC`, a high-compression variant of LZ4. It compresses more aggressively than LZ4 but decompresses at the same speed:

```sql
CREATE TABLE audit_logs
(
    log_id   UInt64  CODEC(LZ4HC(9)),
    message  String  CODEC(LZ4HC(9)),
    ts       DateTime CODEC(LZ4)
)
ENGINE = MergeTree()
ORDER BY (ts, log_id);
```

`LZ4HC(level)` accepts levels 1-12. Level 9 is a good balance for archival columns.

## Mixing LZ4 with Other Codecs

LZ4 is commonly used as the final stage in a codec chain. For example, apply Delta encoding first (to reduce entropy) then LZ4 for compression:

```sql
CREATE TABLE metrics
(
    metric_id  UInt32   CODEC(LZ4),
    value      Float64  CODEC(Gorilla, LZ4),
    ts         DateTime CODEC(Delta(4), LZ4)
)
ENGINE = MergeTree()
ORDER BY (ts, metric_id);
```

## Measuring LZ4 Performance

Insert a large dataset and compare query times:

```sql
CREATE TABLE bench_lz4
(
    id      UInt64   CODEC(LZ4),
    val     Float64  CODEC(LZ4),
    label   String   CODEC(LZ4),
    ts      DateTime CODEC(LZ4)
)
ENGINE = MergeTree()
ORDER BY ts;

INSERT INTO bench_lz4
SELECT
    number,
    rand() / 1e9,
    ['info','warn','error'][1 + rand() % 3],
    now() - number
FROM numbers(10000000);
```

Query and observe timing:

```sql
SELECT avg(val), count()
FROM bench_lz4
WHERE ts > now() - INTERVAL 7 DAY;
```

Check storage:

```sql
SELECT
    formatReadableSize(sum(data_compressed_bytes))   AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio
FROM system.parts
WHERE active = 1
  AND table = 'bench_lz4'
  AND database = currentDatabase();
```

## When to Choose LZ4

Choose LZ4 when:

- You have high write throughput requirements (millions of rows per second)
- Queries are CPU-bound, not I/O-bound
- Decompression latency is critical for interactive dashboards
- You are already applying a pre-compression codec (Delta, Gorilla) that reduces entropy before LZ4

Choose ZSTD when:

- Storage cost is a priority
- Write throughput is moderate and CPU headroom is available
- Columns contain long, repetitive strings (logs, JSON payloads)

## Changing an Existing Column to LZ4

```sql
ALTER TABLE audit_logs
    MODIFY COLUMN message String CODEC(LZ4);

OPTIMIZE TABLE audit_logs FINAL;
```

## Summary

LZ4 is ClickHouse's fastest compression codec and the default for good reason. Its decompression speed keeps queries fast under heavy I/O load. Use it explicitly when speed is paramount, or combine it with entropy-reducing codecs like Delta and Gorilla to get both small file sizes and rapid decompression.
