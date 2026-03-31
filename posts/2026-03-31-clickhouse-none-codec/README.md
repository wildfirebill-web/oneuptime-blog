# How to Use NONE Codec in ClickHouse and When It Makes Sense

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Compression, Database, Performance, Storage, Optimization

Description: Learn when and how to use the NONE codec in ClickHouse to disable compression on specific columns for maximum read speed and reduced CPU usage.

---

ClickHouse compresses all column data by default using LZ4. The NONE codec explicitly disables compression, storing raw bytes on disk. This sounds counterproductive, but there are legitimate scenarios where uncompressed storage is the right choice. Understanding those scenarios helps you build tables that are optimally tuned for their access patterns.

## What NONE Does

`CODEC(NONE)` tells ClickHouse to store the column data without any compression or transform. Reads return raw bytes without a decompression step, eliminating CPU overhead entirely during scans.

```sql
CREATE TABLE raw_events
(
    event_id   UInt64  CODEC(NONE),
    event_type UInt8   CODEC(NONE),
    ts         DateTime CODEC(NONE)
)
ENGINE = MergeTree()
ORDER BY ts;
```

## When NONE Makes Sense

### Already-Compressed Data

If a column stores externally compressed blobs (gzip, brotli, images, PDFs), applying LZ4 or ZSTD adds CPU overhead without reducing size. The data is already incompressible:

```sql
CREATE TABLE document_store
(
    doc_id    UInt64  CODEC(LZ4),
    file_name String  CODEC(LZ4),
    -- Store pre-compressed binary payload without re-compression
    content   String  CODEC(NONE)
)
ENGINE = MergeTree()
ORDER BY doc_id;
```

### CPU-Constrained Ingestion

On servers where write throughput is limited by CPU and storage is cheap (NVMe SSD arrays), NONE eliminates the compression cost on hot write paths:

```sql
CREATE TABLE raw_ingest_buffer
(
    line     String   CODEC(NONE),
    ts       DateTime CODEC(NONE)
)
ENGINE = MergeTree()
ORDER BY ts
TTL ts + INTERVAL 1 HOUR;
```

After TTL expires or after a materialized view transforms the data, richer compression applies downstream.

### In-Memory or Ramdisk Tables

For temporary computation tables stored on a RAM filesystem, disk space is not a concern:

```sql
CREATE TABLE temp_aggregation
(
    group_key UInt64 CODEC(NONE),
    agg_sum   Int64  CODEC(NONE),
    agg_cnt   UInt32 CODEC(NONE)
)
ENGINE = MergeTree()
ORDER BY group_key
SETTINGS storage_policy = 'ram';
```

### High-Entropy Small Tables

Tables with fewer than a few thousand rows rarely benefit from compression. The overhead of tracking compressed block metadata can exceed the space savings:

```sql
CREATE TABLE config_values
(
    key   String CODEC(NONE),
    value String CODEC(NONE)
)
ENGINE = MergeTree()
ORDER BY key;
```

## Comparing NONE vs LZ4 on Incompressible Data

```sql
CREATE TABLE binary_none
(
    id   UInt64 CODEC(LZ4),
    blob String CODEC(NONE)
)
ENGINE = MergeTree()
ORDER BY id;

CREATE TABLE binary_lz4
(
    id   UInt64 CODEC(LZ4),
    blob String CODEC(LZ4)
)
ENGINE = MergeTree()
ORDER BY id;

-- Insert random 256-byte strings (incompressible)
INSERT INTO binary_none SELECT number, hex(randomString(128)) FROM numbers(1000000);
INSERT INTO binary_lz4  SELECT number, hex(randomString(128)) FROM numbers(1000000);
```

Check that LZ4 adds no benefit (and may slightly inflate size due to framing overhead):

```sql
SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes))   AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio
FROM system.parts
WHERE active = 1
  AND table IN ('binary_none', 'binary_lz4')
  AND database = currentDatabase()
GROUP BY table
ORDER BY table;
```

## When NOT to Use NONE

Avoid NONE for:

- String columns with repetitive text (logs, JSON, user agents)
- Timestamp columns (Delta + LZ4 can achieve 5-10x compression)
- Integer columns with sequential values
- Any column you expect to filter or aggregate at scale

Using NONE on compressible data significantly increases storage costs and reduces I/O cache efficiency.

## Mixing NONE with Other Codecs

You can use NONE on specific columns while keeping compression on others:

```sql
CREATE TABLE mixed_codecs
(
    id          UInt64   CODEC(Delta(8), LZ4),
    ts          DateTime CODEC(DoubleDelta, LZ4),
    description String   CODEC(ZSTD(3)),
    -- Pre-compressed binary attachment
    attachment  String   CODEC(NONE)
)
ENGINE = MergeTree()
ORDER BY (ts, id);
```

## Verifying NONE in System Tables

```sql
SELECT name, compression_codec
FROM system.columns
WHERE table = 'raw_events'
  AND database = currentDatabase();
```

## Summary

NONE is not a default you should apply broadly. It is a precise tool for specific situations: pre-compressed payloads, CPU-constrained ingestion pipelines, and temporary in-memory tables. For everything else, LZ4 or ZSTD with appropriate transform codecs will reduce your storage costs and improve I/O efficiency.
