# How to Use DoubleDelta Codec in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Compression, DoubleDelta, TimeSeries, Database, Performance

Description: Learn how to use the DoubleDelta codec in ClickHouse to compress timestamps and integer columns with constant-rate growth for maximum storage savings.

---

DoubleDelta is a second-order delta transform codec in ClickHouse. Where Delta stores first differences (value[i] - value[i-1]), DoubleDelta stores second differences (delta[i] - delta[i-1]). When data grows at a constant rate, first differences are a constant, and second differences become zero. Near-zero values compress to almost nothing, making DoubleDelta exceptional for regularly-spaced timestamps.

## How DoubleDelta Works

Consider timestamps spaced exactly 10 seconds apart:

```text
Original:     1000, 1010, 1020, 1030, 1040
Delta:        1000,   10,   10,   10,   10
DoubleDelta:  1000,   10,    0,    0,    0
```

The second differences are all zero. Even when spacing is not perfectly constant, DoubleDelta produces very small residuals that compress far better than raw values.

This algorithm was introduced in the Gorilla paper (Facebook, 2015) for timestamps and is well-suited to any monotonically increasing column with stable growth.

## Syntax

```sql
CODEC(DoubleDelta, LZ4)
CODEC(DoubleDelta, ZSTD(3))
```

DoubleDelta is a pure transform and must be followed by a compressor.

## Applying DoubleDelta to Timestamps

```sql
CREATE TABLE sensor_readings
(
    sensor_id  UInt32      CODEC(LZ4),
    value      Float32     CODEC(Gorilla, LZ4),
    ts         DateTime    CODEC(DoubleDelta, LZ4)
)
ENGINE = MergeTree()
ORDER BY (sensor_id, ts);
```

For high-resolution timestamps:

```sql
CREATE TABLE high_res_events
(
    event_id  UInt64        CODEC(DoubleDelta, LZ4),
    payload   String        CODEC(ZSTD(3)),
    ts        DateTime64(6) CODEC(DoubleDelta, LZ4)
)
ENGINE = MergeTree()
ORDER BY (ts, event_id);
```

## DoubleDelta vs Delta: When to Choose Which

| Data pattern | Best codec |
|--------------|------------|
| Irregular spacing, large gaps | Delta |
| Regular spacing (fixed intervals) | DoubleDelta |
| Accelerating growth (non-linear) | Delta |
| Monotonic with consistent rate | DoubleDelta |

## Benchmarking DoubleDelta

```sql
CREATE TABLE ts_delta_bench
(
    ts   DateTime CODEC(Delta(4), LZ4),
    val  UInt32   CODEC(Delta(4), LZ4)
)
ENGINE = MergeTree()
ORDER BY ts;

CREATE TABLE ts_doubledelta_bench
(
    ts   DateTime CODEC(DoubleDelta, LZ4),
    val  UInt32   CODEC(DoubleDelta, LZ4)
)
ENGINE = MergeTree()
ORDER BY ts;

-- Regular 1-second intervals
INSERT INTO ts_delta_bench
SELECT
    toDateTime('2024-01-01') + number,
    1000 + number / 100
FROM numbers(10000000);

INSERT INTO ts_doubledelta_bench
SELECT
    toDateTime('2024-01-01') + number,
    1000 + number / 100
FROM numbers(10000000);
```

Check compression:

```sql
SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes))   AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio
FROM system.parts
WHERE active = 1
  AND table IN ('ts_delta_bench', 'ts_doubledelta_bench')
  AND database = currentDatabase()
GROUP BY table
ORDER BY table;
```

On uniformly-spaced data, DoubleDelta typically achieves 20-40% better compression than Delta.

## Irregular Timestamps and DoubleDelta

When event timestamps vary, DoubleDelta residuals grow larger and the advantage disappears. Test with irregular data:

```sql
CREATE TABLE ts_irregular
(
    ts  DateTime CODEC(DoubleDelta, LZ4)
)
ENGINE = MergeTree()
ORDER BY ts;

INSERT INTO ts_irregular
SELECT now() - (rand() % 86400)
FROM numbers(5000000);
```

Compare this against a Delta-encoded version and pick the one with the better ratio.

## Combining DoubleDelta with ZSTD

Pairing DoubleDelta with ZSTD can squeeze out additional bytes:

```sql
CREATE TABLE compressed_ts
(
    ts  DateTime CODEC(DoubleDelta, ZSTD(3))
)
ENGINE = MergeTree()
ORDER BY ts;
```

The additional gain is usually small because DoubleDelta already reduces the values to near-zero, leaving little entropy for ZSTD to exploit. Use LZ4 after DoubleDelta unless storage is extremely constrained.

## Real-World Example: IoT Pipeline

```sql
CREATE TABLE iot_telemetry
(
    device_id    UInt32       CODEC(LZ4),
    temperature  Float32      CODEC(Gorilla, LZ4),
    humidity     Float32      CODEC(Gorilla, LZ4),
    pressure     Float32      CODEC(Gorilla, LZ4),
    ts           DateTime64(3) CODEC(DoubleDelta, LZ4)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (device_id, ts)
TTL ts + INTERVAL 1 YEAR DELETE;
```

## Summary

DoubleDelta is the optimal codec for timestamps and other integer sequences that grow at a constant rate. It reduces second-order differences to near-zero values that compress far more efficiently than Delta alone. Use it for regular time-series workloads with consistent sampling intervals, and always follow it with LZ4 or ZSTD.
