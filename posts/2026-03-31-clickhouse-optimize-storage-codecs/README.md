# How to Optimize Storage Size with Column Codecs in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Compression, Storage, Codec, Optimization

Description: Learn how to optimize ClickHouse storage size by selecting the right column codecs based on data type, cardinality, and access patterns.

---

## Why Column Codec Selection Matters

ClickHouse's columnar storage means each column can be independently compressed. Choosing the right codec per column can reduce storage by 5-20x, lowering costs and improving cache efficiency.

## Audit Your Current Storage

Start by finding the biggest tables and columns:

```sql
SELECT
    table,
    column,
    type,
    compression_codec,
    formatReadableSize(data_compressed_bytes) AS compressed,
    formatReadableSize(data_uncompressed_bytes) AS uncompressed,
    round(data_uncompressed_bytes / data_compressed_bytes, 2) AS ratio
FROM system.columns
WHERE database = 'default'
ORDER BY data_compressed_bytes DESC
LIMIT 20;
```

## Codec Recommendations by Data Type

### Timestamps and monotonic integers

```sql
ts DateTime CODEC(Delta(4), LZ4)
ts DateTime64(3) CODEC(Delta(8), LZ4)
event_id UInt64 CODEC(Delta(8), ZSTD(1))
```

### Slowly changing values

```sql
offset UInt64 CODEC(DoubleDelta, ZSTD(1))
```

### Float time-series

```sql
cpu_usage Float64 CODEC(Gorilla, ZSTD(1))
temperature Float32 CODEC(Gorilla, LZ4)
```

### High-cardinality strings

```sql
message String CODEC(ZSTD(3))
```

### Low-cardinality strings

```sql
status LowCardinality(String)  -- no extra codec needed, dictionary encoding applies
```

### Random data (UUIDs, hashes)

```sql
request_id UUID CODEC(LZ4)      -- minimal benefit, low overhead
```

## Apply Codecs to Existing Table

```sql
ALTER TABLE events
    MODIFY COLUMN ts DateTime CODEC(Delta(4), LZ4),
    MODIFY COLUMN value Float64 CODEC(Gorilla, ZSTD(1)),
    MODIFY COLUMN user_id UInt32 CODEC(Delta(4), LZ4),
    MODIFY COLUMN message String CODEC(ZSTD(3));
```

Changes apply to new data parts. Force merge to apply to all data:

```sql
OPTIMIZE TABLE events FINAL;
```

## LowCardinality for String Columns

```sql
ALTER TABLE events
MODIFY COLUMN event_type LowCardinality(String);
```

LowCardinality uses dictionary encoding and reduces string storage dramatically for columns with under 10,000 unique values.

## Measure Results

```sql
SELECT
    column,
    compression_codec,
    formatReadableSize(data_compressed_bytes) AS compressed,
    round(data_uncompressed_bytes / data_compressed_bytes, 2) AS ratio
FROM system.columns
WHERE table = 'events'
ORDER BY data_compressed_bytes DESC;
```

## ZSTD Level Trade-offs

```text
ZSTD(1)  - fast compression, good ratio (recommended default)
ZSTD(3)  - slower, better ratio for strings and text
ZSTD(9)  - very slow, best ratio for cold archival data
LZ4      - fastest, lower ratio, best for hot data
LZ4HC(9) - slower, better ratio than LZ4
```

## Disk Space vs. Query Speed

Higher compression ratios mean less data read from disk, which can speed up I/O-bound queries. For CPU-bound queries, additional decompression overhead may slow things down slightly. Test both before committing.

## Summary

Optimizing ClickHouse storage starts with auditing `system.columns` for compression ratios, then applying codec recommendations by data type. Delta and Gorilla pre-processors dramatically improve compression for time-series and float data. LowCardinality handles repetitive strings. Run `OPTIMIZE FINAL` to apply changes across all existing data parts.
