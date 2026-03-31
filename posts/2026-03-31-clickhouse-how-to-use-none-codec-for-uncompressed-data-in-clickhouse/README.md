# How to Use NONE Codec for Uncompressed Data in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, NONE Codec, Compression, Storage, Performance

Description: Learn when and how to use the NONE codec in ClickHouse to store column data without compression, and understand the trade-offs versus compressed codecs.

---

## What Is the NONE Codec

The `NONE` codec in ClickHouse disables all compression for a column. Data is stored as-is on disk without any encoding. This is useful in specific scenarios where compression overhead is undesirable or where the data is already in a format that does not compress well.

## Applying the NONE Codec

```sql
CREATE TABLE raw_binary_data (
    id          UInt64    CODEC(NONE),
    payload     String    CODEC(NONE),
    ts          DateTime  CODEC(Delta, LZ4)
) ENGINE = MergeTree()
ORDER BY id;
```

## When to Use NONE Codec

**Pre-compressed data**: If you store data that is already compressed (JPEG images, ZIP files, encrypted blobs), applying LZ4 or ZSTD on top adds CPU overhead without reducing size.

```sql
CREATE TABLE encrypted_payloads (
    record_id   UInt64    CODEC(NONE),
    ciphertext  String    CODEC(NONE),  -- Already encrypted/random bytes
    nonce       FixedString(12) CODEC(NONE),
    created_at  DateTime  CODEC(Delta, LZ4)
) ENGINE = MergeTree()
ORDER BY (record_id, created_at);
```

**High-entropy random data**: Random or pseudo-random data does not compress. Using NONE avoids the decompression overhead on reads.

```sql
CREATE TABLE random_tokens (
    token   FixedString(32) CODEC(NONE),  -- 32-byte random token
    user_id UInt64 CODEC(Delta, LZ4),
    ts      DateTime CODEC(Delta, LZ4)
) ENGINE = MergeTree()
ORDER BY ts;
```

**Benchmarking**: Disable compression to isolate raw I/O performance from CPU decompression overhead.

## Comparing NONE vs LZ4 on Compressible Data

```sql
CREATE TABLE test_none (
    data String CODEC(NONE)
) ENGINE = MergeTree() ORDER BY tuple();

CREATE TABLE test_lz4 (
    data String CODEC(LZ4)
) ENGINE = MergeTree() ORDER BY tuple();

-- Insert the same compressible data
INSERT INTO test_none SELECT toString(number) FROM numbers(1000000);
INSERT INTO test_lz4  SELECT toString(number) FROM numbers(1000000);

SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes)) AS on_disk,
    formatReadableSize(sum(data_uncompressed_bytes)) AS logical_size
FROM system.parts
WHERE database = 'default'
  AND table IN ('test_none', 'test_lz4')
  AND active = 1
GROUP BY table;
```

## Checking Which Columns Use NONE

```sql
SELECT
    column,
    type,
    compression_codec
FROM system.columns
WHERE database = 'default'
  AND table = 'raw_binary_data'
  AND compression_codec LIKE '%NONE%';
```

## Reverting to Default Compression

To switch a NONE-coded column back to the default:

```sql
ALTER TABLE raw_binary_data
    MODIFY COLUMN payload String CODEC(LZ4);

-- Recompress data in existing parts
OPTIMIZE TABLE raw_binary_data FINAL;
```

## NONE Codec vs Not Specifying a Codec

Not specifying a codec uses the server default (usually LZ4). Explicitly setting `CODEC(NONE)` ensures no compression is ever applied, even if the server default changes.

```sql
-- These are different:
data1 String,              -- Uses server default codec
data2 String CODEC(NONE),  -- Always uncompressed
data3 String CODEC(LZ4)    -- Always LZ4
```

## Use with External Disk or Object Storage

When tiering data to S3 or GCS where objects are already deduplicated and stored efficiently, NONE codec can reduce CPU overhead on reads:

```sql
CREATE TABLE archived_events ON CLUSTER cluster (
    ts          DateTime      CODEC(Delta, LZ4),
    event_data  String        CODEC(NONE)  -- Pre-serialized binary, not compressible
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/archived_events', '{replica}')
ORDER BY ts
TTL ts + INTERVAL 365 DAY TO DISK 's3_disk';
```

## Summary

The `NONE` codec disables column compression in ClickHouse and is best used for pre-compressed blobs, encrypted data, or high-entropy random values where compression would waste CPU without saving space. For most workloads with structured data, LZ4 or ZSTD will provide better performance. Use `system.columns` to audit codec assignments and `OPTIMIZE TABLE FINAL` to recompress existing parts after changing codecs.
