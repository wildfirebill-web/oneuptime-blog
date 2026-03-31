# How to Use Gorilla Codec in ClickHouse for Float Data

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Gorilla Codec, Float Compression, Time-Series, Storage

Description: Learn how to use the Gorilla codec in ClickHouse for efficient compression of floating-point time-series data, reducing storage while preserving precision.

---

## What Is the Gorilla Codec

The Gorilla codec is based on Facebook's Gorilla time-series database compression algorithm. It stores XOR differences between consecutive floating-point values. When floats change gradually (as metrics typically do), successive XOR values are small and compress well.

This makes Gorilla ideal for float columns in metrics, sensor data, financial time series, and monitoring data.

## Applying the Gorilla Codec

```sql
CREATE TABLE server_metrics (
    ts          DateTime    CODEC(Delta, LZ4),
    host        String      CODEC(LZ4),
    cpu_usage   Float32     CODEC(Gorilla, LZ4),
    memory_pct  Float32     CODEC(Gorilla, LZ4),
    disk_io     Float64     CODEC(Gorilla, LZ4),
    net_bytes   Float64     CODEC(Gorilla, LZ4)
) ENGINE = MergeTree()
ORDER BY (host, ts);
```

Always pair Gorilla with `LZ4` or `ZSTD` as Gorilla alone does not produce byte-aligned output that typical compressors handle optimally.

## Measuring Gorilla vs Raw Float Compression

```sql
CREATE TABLE metrics_no_gorilla (
    ts    DateTime,
    value Float64
) ENGINE = MergeTree() ORDER BY ts;

CREATE TABLE metrics_gorilla (
    ts    DateTime  CODEC(Delta, LZ4),
    value Float64   CODEC(Gorilla, LZ4)
) ENGINE = MergeTree() ORDER BY ts;

-- Generate realistic slowly-changing float data
INSERT INTO metrics_no_gorilla
    SELECT now() - number, 50 + sin(number * 0.001) * 10
    FROM numbers(10000000);

INSERT INTO metrics_gorilla
    SELECT now() - number, 50 + sin(number * 0.001) * 10
    FROM numbers(10000000);

SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio
FROM system.parts
WHERE database = 'default'
  AND table IN ('metrics_no_gorilla', 'metrics_gorilla')
  AND active = 1
GROUP BY table;
```

## IoT Sensor Data Example

```sql
CREATE TABLE iot_sensors (
    sensor_id   UInt32      CODEC(Delta, LZ4),
    ts          DateTime64(3) CODEC(Delta, LZ4),
    temperature Float32     CODEC(Gorilla, LZ4),
    humidity    Float32     CODEC(Gorilla, LZ4),
    pressure    Float64     CODEC(Gorilla, LZ4),
    voltage     Float32     CODEC(Gorilla, LZ4)
) ENGINE = MergeTree()
ORDER BY (sensor_id, ts);
```

## Financial Price Data Example

```sql
CREATE TABLE stock_prices (
    symbol      String      CODEC(LZ4),
    ts          DateTime    CODEC(Delta, LZ4),
    open_price  Float64     CODEC(Gorilla, LZ4),
    high_price  Float64     CODEC(Gorilla, LZ4),
    low_price   Float64     CODEC(Gorilla, LZ4),
    close_price Float64     CODEC(Gorilla, LZ4),
    volume      UInt64      CODEC(Delta, LZ4)
) ENGINE = MergeTree()
ORDER BY (symbol, ts);
```

## When Gorilla Helps vs When It Doesn't

Gorilla is most effective when:
- Float values change gradually between consecutive rows
- Data is sorted so nearby rows have related values
- Values don't jump randomly between samples

Gorilla is less effective when:
- Float values are random or have large jumps
- Rows are not ordered by the time dimension
- Values are integers stored as floats

## Altering an Existing Table

```sql
ALTER TABLE server_metrics
    MODIFY COLUMN cpu_usage Float32 CODEC(Gorilla, LZ4);

-- Recompress existing data
OPTIMIZE TABLE server_metrics FINAL;
```

## Verifying Codec Configuration

```sql
SELECT
    column,
    type,
    compression_codec
FROM system.columns
WHERE database = 'default'
  AND table = 'server_metrics';
```

## Combining Gorilla with DoubleDelta for Maximum Compression

For very dense time series with slow-moving floats:

```sql
CREATE TABLE dense_metrics (
    ts_ms   UInt64   CODEC(DoubleDelta, LZ4),
    value   Float64  CODEC(Gorilla, LZ4)
) ENGINE = MergeTree()
ORDER BY ts_ms;
```

## Summary

The Gorilla codec provides efficient lossless compression for floating-point columns in ClickHouse by exploiting temporal locality in metric data. Pair it with `LZ4` or `ZSTD` as a secondary codec, and use `Delta` or `DoubleDelta` on your timestamp columns to get maximum storage reduction for time-series workloads. Gorilla typically reduces float column storage by 2-5x compared to raw encoding with LZ4 alone.
