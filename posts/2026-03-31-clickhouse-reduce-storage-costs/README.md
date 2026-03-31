# How to Reduce ClickHouse Storage Costs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Storage, Cost Optimization, Compression, TTL

Description: Learn practical strategies to reduce ClickHouse storage costs by optimizing compression, applying TTL policies, and choosing the right data types.

---

## Why Storage Costs Matter

ClickHouse can ingest enormous volumes of data. Without active management, storage costs grow linearly with data volume. The good news is that ClickHouse provides multiple overlapping tools to dramatically reduce storage without losing query capability.

## 1. Choose the Right Compression Codec

The default LZ4 codec balances speed and compression ratio. For cold or infrequently queried data, ZSTD provides better compression:

```sql
ALTER TABLE events MODIFY COLUMN
    request_body String CODEC(ZSTD(3));
```

For time-series data, delta encoding followed by compression can be very effective:

```sql
ALTER TABLE metrics MODIFY COLUMN
    value Float64 CODEC(Delta, ZSTD);

ALTER TABLE metrics MODIFY COLUMN
    timestamp DateTime CODEC(DoubleDelta, ZSTD);
```

## 2. Measure Current Compression Ratio

```sql
SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio
FROM system.columns
WHERE database = 'default'
GROUP BY table
ORDER BY sum(data_compressed_bytes) DESC;
```

## 3. Use Low-Cardinality for String Columns

```sql
ALTER TABLE events MODIFY COLUMN event_type LowCardinality(String);
ALTER TABLE events MODIFY COLUMN status_code LowCardinality(String);
```

`LowCardinality` encodes repeated values as integers, reducing storage by 3-10x for columns with fewer than 10,000 distinct values.

## 4. Set TTL Policies to Delete Old Data

```sql
ALTER TABLE events
    MODIFY TTL created_at + INTERVAL 90 DAY;
```

Data older than 90 days is automatically deleted during background merges.

Run TTL cleanup immediately:

```sql
OPTIMIZE TABLE events FINAL;
```

## 5. Tiered Storage (Hot to Cold)

Move old data to cheaper storage automatically:

```sql
ALTER TABLE events
    MODIFY TTL
        created_at + INTERVAL 7 DAY TO DISK 'cold_storage',
        created_at + INTERVAL 90 DAY DELETE;
```

Configure `cold_storage` in `storage_policy.xml` pointing to an S3 bucket or slower disk.

## 6. Use Appropriate Data Types

Choose the smallest type that fits your data:

```text
Use UInt8 (0-255) instead of UInt64 for small integers
Use Float32 instead of Float64 when precision allows
Use Date instead of DateTime when time precision is not needed
Use FixedString(N) instead of String for fixed-length values
```

```sql
-- Before
latency_ms UInt64,    -- 8 bytes
-- After
latency_ms UInt32,    -- 4 bytes (max ~4.2 billion ms)
```

## 7. Drop Unused Columns

```sql
ALTER TABLE events DROP COLUMN legacy_field;
ALTER TABLE events DROP COLUMN deprecated_metadata;
```

## 8. Review Partition Granularity

Partitioning by day instead of month can increase part count and overhead:

```sql
-- Consider monthly partitioning for high-volume tables
PARTITION BY toYYYYMM(created_at)
-- instead of daily
PARTITION BY toDate(created_at)
```

## Summary

Reduce ClickHouse storage costs by switching to ZSTD compression, using `LowCardinality` for string columns, applying TTL policies to automatically delete or tier old data, and choosing the smallest appropriate data types. Measure your current compression ratio in `system.columns` to identify which tables have the most room for improvement.
