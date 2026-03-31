# How to Choose the Right Compression Codec in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Compression, Codec, Database, Performance, Optimization

Description: A practical guide to selecting the right ClickHouse compression codec for every column type, with benchmarks and decision trees for production schemas.

---

ClickHouse provides a rich set of compression codecs, and picking the wrong one can leave significant storage savings on the table or slow down ingestion. This guide gives you a decision-making framework, column-type recommendations, and SQL benchmarks to validate your choices.

## The Codec Menu

| Codec | Type | Best for |
|-------|------|---------|
| LZ4 | Compressor | Default; fast decompression |
| LZ4HC | Compressor | Better ratio than LZ4, same decompression speed |
| ZSTD(level) | Compressor | High ratio; flexible CPU/ratio trade-off |
| Delta | Transform | Sequential integers, timestamps |
| DoubleDelta | Transform | Evenly-spaced timestamps |
| Gorilla | Transform | Smooth float series |
| T64 | Transform | Clustered integers, enums |
| NONE | Bypass | Pre-compressed blobs, RAM tables |

## Decision Tree

```text
What type is the column?
|
+-- Float (Float32, Float64)
|   +-- Values are smooth/correlated?   -> Gorilla + LZ4
|   +-- Values are random?              -> ZSTD(3)
|
+-- DateTime / DateTime64
|   +-- Regular sampling intervals?    -> DoubleDelta + LZ4
|   +-- Irregular / event-driven?      -> Delta(4) + LZ4
|
+-- Integer (monotonic, e.g., row ID)  -> Delta + LZ4
+-- Integer (clustered range)          -> T64 + LZ4
+-- Integer (low-cardinality enum)     -> T64 + ZSTD(3)
|
+-- String
|   +-- Repetitive text (logs, JSON)?  -> ZSTD(3)
|   +-- Random / UUIDs?                -> ZSTD(3) or LZ4
|   +-- Pre-compressed binary?         -> NONE
|
+-- Small table (< 10k rows)           -> LZ4 (overhead not worth it)
```

## Codec Comparison Table

```sql
-- Create reference tables to benchmark each codec type
CREATE TABLE codec_bench_lz4       (ts DateTime CODEC(LZ4),              val Float64 CODEC(LZ4))        ENGINE = MergeTree() ORDER BY ts;
CREATE TABLE codec_bench_zstd      (ts DateTime CODEC(ZSTD(3)),          val Float64 CODEC(ZSTD(3)))     ENGINE = MergeTree() ORDER BY ts;
CREATE TABLE codec_bench_delta_lz4 (ts DateTime CODEC(Delta(4), LZ4),    val Float64 CODEC(LZ4))        ENGINE = MergeTree() ORDER BY ts;
CREATE TABLE codec_bench_optimal   (ts DateTime CODEC(DoubleDelta, LZ4), val Float64 CODEC(Gorilla, LZ4)) ENGINE = MergeTree() ORDER BY ts;

-- Same dataset into all four
INSERT INTO codec_bench_lz4       SELECT toDateTime('2024-01-01') + number, 50 + 5 * sin(number / 100.0) FROM numbers(10000000);
INSERT INTO codec_bench_zstd      SELECT toDateTime('2024-01-01') + number, 50 + 5 * sin(number / 100.0) FROM numbers(10000000);
INSERT INTO codec_bench_delta_lz4 SELECT toDateTime('2024-01-01') + number, 50 + 5 * sin(number / 100.0) FROM numbers(10000000);
INSERT INTO codec_bench_optimal   SELECT toDateTime('2024-01-01') + number, 50 + 5 * sin(number / 100.0) FROM numbers(10000000);
```

Measure ratios:

```sql
SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes))   AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio
FROM system.parts
WHERE active = 1
  AND table LIKE 'codec_bench_%'
  AND database = currentDatabase()
GROUP BY table
ORDER BY ratio DESC;
```

## Recommended Codecs by Column Role

### Timestamp columns

```sql
ts DateTime     CODEC(DoubleDelta, LZ4)    -- regular intervals
ts DateTime     CODEC(Delta(4), LZ4)       -- irregular events
ts DateTime64(3) CODEC(DoubleDelta, LZ4)   -- millisecond precision
```

### Auto-increment primary key

```sql
id UInt64 CODEC(Delta(8), LZ4)
```

### Float metrics

```sql
cpu_pct    Float32 CODEC(Gorilla, LZ4)
latency_ms Float64 CODEC(Gorilla, LZ4)
```

### Low-cardinality status/enum columns

```sql
status     UInt8  CODEC(T64, LZ4)
level      UInt16 CODEC(T64, ZSTD(3))
```

### Log messages and JSON

```sql
message String CODEC(ZSTD(3))
json    String CODEC(ZSTD(6))
```

### Pre-compressed binary

```sql
blob String CODEC(NONE)
```

## Production Schema Example

```sql
CREATE TABLE production_events
(
    event_id    UInt64        CODEC(Delta(8), LZ4),
    service_id  UInt16        CODEC(T64, LZ4),
    level       UInt8         CODEC(T64, LZ4),
    latency_ms  Float32       CODEC(Gorilla, LZ4),
    error_rate  Float32       CODEC(Gorilla, LZ4),
    message     String        CODEC(ZSTD(3)),
    metadata    String        CODEC(ZSTD(6)),
    ts          DateTime64(3) CODEC(DoubleDelta, LZ4)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (service_id, ts, event_id)
TTL ts + INTERVAL 90 DAY DELETE;
```

## Validating Your Choices

Always validate codec choices against real data. Use `system.parts` per-column compression stats:

```sql
SELECT
    name,
    type,
    compression_codec,
    formatReadableSize(data_compressed_bytes)   AS compressed,
    formatReadableSize(data_uncompressed_bytes) AS uncompressed,
    round(data_uncompressed_bytes / data_compressed_bytes, 2) AS ratio
FROM system.columns
WHERE table = 'production_events'
  AND database = currentDatabase()
ORDER BY ratio ASC;
```

Columns with a ratio close to 1.0 may need a different codec.

## CPU vs Storage Trade-off

| Priority | Recommended approach |
|----------|---------------------|
| Fastest writes | LZ4 on all columns |
| Fastest reads | LZ4 (less decompression work) |
| Minimum storage | ZSTD(6-9) + appropriate transforms |
| Balanced | ZSTD(3) + transforms |

## Summary

The right codec depends on the column type, value distribution, and your workload priorities. Start with the decision tree above, apply domain-specific transforms, follow with LZ4 or ZSTD, and always validate with `system.parts`. A well-tuned codec strategy can reduce storage by 5-20x compared to using plain LZ4 everywhere.
