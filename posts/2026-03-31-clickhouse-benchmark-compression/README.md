# How to Benchmark Compression Ratios in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Compression, Benchmark, Storage, Performance

Description: Learn how to benchmark and compare compression ratios for different codecs in ClickHouse to select the best option for your data types and query patterns.

---

## Why Benchmark Compression

Different codecs perform differently depending on data distribution. Benchmarking before choosing ensures you get the best storage efficiency without sacrificing query speed.

## Create Test Tables for Comparison

```sql
CREATE TABLE test_lz4 (
    ts DateTime CODEC(LZ4),
    value Float64 CODEC(LZ4),
    host String CODEC(LZ4)
) ENGINE = MergeTree() ORDER BY ts;

CREATE TABLE test_zstd (
    ts DateTime CODEC(ZSTD(1)),
    value Float64 CODEC(ZSTD(1)),
    host String CODEC(ZSTD(1))
) ENGINE = MergeTree() ORDER BY ts;

CREATE TABLE test_delta_lz4 (
    ts DateTime CODEC(Delta(4), LZ4),
    value Float64 CODEC(Gorilla, LZ4),
    host String CODEC(LZ4)
) ENGINE = MergeTree() ORDER BY ts;

CREATE TABLE test_delta_zstd (
    ts DateTime CODEC(Delta(4), ZSTD(1)),
    value Float64 CODEC(Gorilla, ZSTD(1)),
    host String CODEC(ZSTD(1))
) ENGINE = MergeTree() ORDER BY ts;
```

## Load Identical Data

```sql
INSERT INTO test_lz4 SELECT ts, value, host FROM source_data;
INSERT INTO test_zstd SELECT ts, value, host FROM source_data;
INSERT INTO test_delta_lz4 SELECT ts, value, host FROM source_data;
INSERT INTO test_delta_zstd SELECT ts, value, host FROM source_data;
```

## Compare Storage Sizes

```sql
SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio
FROM system.columns
WHERE table IN ('test_lz4', 'test_zstd', 'test_delta_lz4', 'test_delta_zstd')
GROUP BY table
ORDER BY ratio DESC;
```

## Compare Per-Column Compression

```sql
SELECT
    table,
    column,
    formatReadableSize(data_compressed_bytes) AS compressed,
    round(data_uncompressed_bytes / data_compressed_bytes, 2) AS ratio
FROM system.columns
WHERE table LIKE 'test_%'
ORDER BY table, column;
```

## Benchmark Query Speed

Use `clickhouse-benchmark` to measure query time:

```bash
echo "SELECT sum(value) FROM test_lz4 WHERE ts BETWEEN '2026-01-01' AND '2026-02-01'" \
  | clickhouse-benchmark --iterations=100 --concurrency=4
```

Repeat for each table and compare median and 99th percentile times.

## Measure Disk Parts

```sql
SELECT
    table,
    formatReadableSize(sum(bytes_on_disk)) AS disk_size
FROM system.parts
WHERE active = 1
  AND table IN ('test_lz4', 'test_zstd', 'test_delta_lz4', 'test_delta_zstd')
GROUP BY table
ORDER BY sum(bytes_on_disk);
```

## Force Merges Before Measuring

```sql
OPTIMIZE TABLE test_lz4 FINAL;
OPTIMIZE TABLE test_zstd FINAL;
OPTIMIZE TABLE test_delta_lz4 FINAL;
OPTIMIZE TABLE test_delta_zstd FINAL;
```

Unmerged parts may skew compression measurements.

## Typical Results for Time-Series Data

```text
Codec                  | Ratio
-----------------------|-------
NONE                   | 1.0x
LZ4                    | 3-5x
ZSTD(1)               | 4-8x
Delta + LZ4            | 6-12x
Gorilla + ZSTD(1)      | 8-20x
```

## Summary

Benchmarking ClickHouse compression requires creating identical tables with different codecs, loading the same data, and comparing storage via `system.columns` and `system.parts`. Always run `OPTIMIZE FINAL` before measuring and test query speed alongside ratio - a slower codec with a better ratio may not be worth it.
