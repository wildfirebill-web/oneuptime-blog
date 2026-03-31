# How to Optimize ClickHouse Compression for Cost Savings

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Compression, Cost Optimization, ZSTD, LZ4

Description: Learn how to optimize ClickHouse column compression codecs to maximize storage savings while balancing query performance and CPU usage.

---

## ClickHouse Compression Fundamentals

ClickHouse compresses data at the column level using pluggable codecs. Each column can have its own codec, allowing you to tune compression per data type and access pattern. The right codec can reduce storage by 5-20x compared to uncompressed data.

## Available Codecs

```text
LZ4         - Default. Fast compression/decompression, moderate ratio.
LZ4HC       - Higher compression than LZ4 at slower write speed.
ZSTD(level) - Best ratio. Levels 1-22 (default 1). Level 3 is a good balance.
Delta       - Stores differences between consecutive values. Best for monotonic data.
DoubleDelta - Second-order differences. Ideal for timestamps.
Gorilla     - XOR compression for floating-point values (time-series metrics).
T64         - Integer compression using bit packing.
```

## Measuring Current Compression

```sql
SELECT
    name AS column_name,
    compression_codec,
    formatReadableSize(data_compressed_bytes) AS compressed,
    formatReadableSize(data_uncompressed_bytes) AS uncompressed,
    round(data_uncompressed_bytes / data_compressed_bytes, 2) AS ratio
FROM system.columns
WHERE database = 'default'
  AND table = 'events'
ORDER BY data_compressed_bytes DESC;
```

## Codec Recommendations by Column Type

### Timestamps

```sql
ALTER TABLE metrics MODIFY COLUMN ts DateTime CODEC(DoubleDelta, ZSTD);
```

DoubleDelta captures the pattern of monotonically increasing timestamps and reduces them to near-zero differences.

### Integer Metrics

```sql
ALTER TABLE metrics MODIFY COLUMN value_int UInt64 CODEC(Delta, ZSTD);
```

### Floating-Point Metrics

```sql
ALTER TABLE metrics MODIFY COLUMN cpu_pct Float64 CODEC(Gorilla, ZSTD);
```

Gorilla is specifically designed for floating-point time series data.

### Low-Cardinality Strings

```sql
ALTER TABLE events MODIFY COLUMN event_type LowCardinality(String);
```

`LowCardinality` is not a codec but an encoding that works like a dictionary - far more effective than compression alone for repeated string values.

### High-Cardinality Strings (UUIDs, hashes)

```sql
ALTER TABLE events MODIFY COLUMN trace_id FixedString(16) CODEC(ZSTD(3));
```

ZSTD works well for UUID-like data. Avoid LZ4 for high-entropy strings as it offers almost no compression.

## Testing Compression Before Applying

Create a test table with different codecs and compare:

```sql
CREATE TABLE events_test_codec AS events
ENGINE = MergeTree()
ORDER BY ts
SETTINGS min_bytes_for_wide_part = 0;

-- Insert a sample
INSERT INTO events_test_codec SELECT * FROM events LIMIT 1000000;

-- Alter a column and compare
ALTER TABLE events_test_codec MODIFY COLUMN value Float64 CODEC(Gorilla, ZSTD);
OPTIMIZE TABLE events_test_codec FINAL;

SELECT
    formatReadableSize(data_compressed_bytes) AS compressed,
    formatReadableSize(data_uncompressed_bytes) AS uncompressed
FROM system.parts
WHERE table = 'events_test_codec' AND active = 1;
```

## Applying Codec Changes to Production

```sql
-- Modify the column codec
ALTER TABLE events MODIFY COLUMN ts DateTime CODEC(DoubleDelta, ZSTD);

-- Force re-compression of existing parts
OPTIMIZE TABLE events FINAL;
```

Note: `OPTIMIZE TABLE FINAL` rewrites all parts and is resource-intensive. Schedule during low-traffic periods.

## Impact on Query Performance

Better compression does not always mean slower queries. Faster decompression means more data fits in the page cache, which can speed up reads. Test with your specific workload:

```sql
-- Measure query time before and after codec change
SET log_queries = 1;
SELECT avg(value) FROM metrics WHERE ts >= now() - INTERVAL 1 DAY;
```

## Summary

Optimize ClickHouse compression by matching codecs to data patterns: `DoubleDelta + ZSTD` for timestamps, `Gorilla + ZSTD` for floating-point metrics, `Delta + ZSTD` for integer sequences, and `LowCardinality` for repeated strings. Measure compression ratios in `system.columns` before and after changes, and always test on a sample before applying to production.
