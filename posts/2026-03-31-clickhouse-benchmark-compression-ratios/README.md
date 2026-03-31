# How to Benchmark Compression Ratios in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Compression, Benchmark, Column Codec, Storage, ZSTD, LZ4

Description: Learn how to benchmark different compression codecs in ClickHouse by comparing on-disk sizes and query times to find the best codec for your data.

---

Choosing the right compression codec in ClickHouse requires benchmarking against your actual data. This guide shows a systematic process to compare codecs using test tables, `OPTIMIZE TABLE`, and `system.columns` queries.

## Benchmark Setup

Create a reference table with real data and no special codec, then clone it with different codecs:

```sql
-- Source table with default codec
CREATE TABLE bench_default AS my_large_table ENGINE = MergeTree() ORDER BY id;
INSERT INTO bench_default SELECT * FROM my_large_table;

-- LZ4 (default)
CREATE TABLE bench_lz4 AS my_large_table ENGINE = MergeTree() ORDER BY id;
ALTER TABLE bench_lz4 MODIFY COLUMN ts    CODEC(LZ4);
ALTER TABLE bench_lz4 MODIFY COLUMN value CODEC(LZ4);
INSERT INTO bench_lz4 SELECT * FROM my_large_table;

-- ZSTD level 3
CREATE TABLE bench_zstd3 AS my_large_table ENGINE = MergeTree() ORDER BY id;
ALTER TABLE bench_zstd3 MODIFY COLUMN ts    CODEC(ZSTD(3));
ALTER TABLE bench_zstd3 MODIFY COLUMN value CODEC(ZSTD(3));
INSERT INTO bench_zstd3 SELECT * FROM my_large_table;

-- Delta + LZ4 (good for timestamps)
CREATE TABLE bench_delta_lz4 AS my_large_table ENGINE = MergeTree() ORDER BY id;
ALTER TABLE bench_delta_lz4 MODIFY COLUMN ts    CODEC(Delta(4), LZ4);
ALTER TABLE bench_delta_lz4 MODIFY COLUMN value CODEC(Gorilla, LZ4);
INSERT INTO bench_delta_lz4 SELECT * FROM my_large_table;
```

## Force Full Merges

Freshly inserted data lives in many small parts that may not reflect final compression. Force a full merge:

```sql
OPTIMIZE TABLE bench_default    FINAL;
OPTIMIZE TABLE bench_lz4        FINAL;
OPTIMIZE TABLE bench_zstd3      FINAL;
OPTIMIZE TABLE bench_delta_lz4  FINAL;
```

## Compare Sizes

```sql
SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes))   AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes))  AS uncompressed,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio
FROM system.parts
WHERE table IN ('bench_default', 'bench_lz4', 'bench_zstd3', 'bench_delta_lz4')
  AND active = 1
GROUP BY table
ORDER BY sum(data_compressed_bytes) ASC
```

## Per-Column Comparison

```sql
SELECT
    table, name,
    formatReadableSize(data_compressed_bytes) AS on_disk,
    round(data_uncompressed_bytes / data_compressed_bytes, 2) AS ratio
FROM system.columns
WHERE table IN ('bench_default', 'bench_lz4', 'bench_zstd3', 'bench_delta_lz4')
ORDER BY table, name
```

## Benchmark Query Speed

Compare read performance for a typical aggregation:

```bash
for t in bench_default bench_lz4 bench_zstd3 bench_delta_lz4; do
  echo "=== $t ==="
  clickhouse-client --query "
    SELECT sum(value), count()
    FROM $t
    WHERE ts >= now() - INTERVAL 7 DAY
  " --time 2>&1 | tail -2
done
```

## Interpreting Results

```text
Codec         Ratio    Read Speed    CPU at Read    Best For
----------    -----    ----------    -----------    --------
LZ4           2-4x     Fast          Low            General purpose
ZSTD(3)       3-6x     Medium        Medium         Text, logs
Delta+LZ4     5-10x    Fast          Low            Timestamps, counters
Gorilla+LZ4   6-12x    Fast          Low            Slowly-changing floats
ZSTD(9)       4-8x     Slower        High           Archive, cold data
```

## Cleanup

```sql
DROP TABLE bench_default;
DROP TABLE bench_lz4;
DROP TABLE bench_zstd3;
DROP TABLE bench_delta_lz4;
```

## Summary

Benchmark ClickHouse compression by creating parallel test tables with different codecs, inserting the same data, running `OPTIMIZE TABLE FINAL`, and comparing sizes via `system.parts` and `system.columns`. Factor in both compression ratio and query latency - a codec that compresses more but reads more slowly may not be the best choice for frequently queried hot data.
