# How to Handle Large Blob Fields During ClickHouse Ingestion

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Blob, Large Payload, Ingestion, String, Compression, Storage

Description: Learn how to handle large blob fields during ClickHouse ingestion, including compression, truncation, and external storage strategies.

---

ClickHouse is optimized for columnar analytics, not blob storage. When ingesting data with large text or binary fields, you need deliberate strategies to avoid performance degradation and excessive storage use.

## The Problem with Large Blobs

ClickHouse stores String columns as contiguous byte sequences. Large blobs in frequently-queried tables cause:

```text
- Slow scans because more data must be read from disk
- High compression CPU overhead for random binary data
- Large part files that slow down merges
- Memory pressure during aggregations
```

## Strategy 1: Truncate at Ingestion

If you only need the first N bytes for search or display, truncate during insert:

```sql
INSERT INTO events (event_id, summary, timestamp)
SELECT
    event_id,
    substring(raw_payload, 1, 1024) AS summary,
    timestamp
FROM events_staging;
```

Or apply length limits as a constraint:

```sql
CREATE TABLE events (
    event_id String,
    payload String,
    timestamp DateTime,
    CONSTRAINT max_payload_size CHECK length(payload) <= 65536
) ENGINE = MergeTree()
ORDER BY (event_id, timestamp);
```

## Strategy 2: Store Blobs in S3, Keep Reference in ClickHouse

Large binaries belong in object storage. Store the S3 URL or key in ClickHouse:

```sql
CREATE TABLE documents (
    doc_id String,
    title String,
    s3_key String,
    content_length UInt64,
    content_type String,
    created_at DateTime
) ENGINE = MergeTree()
ORDER BY (doc_id, created_at);
```

Upload the blob to S3 from your pipeline:

```bash
aws s3 cp /tmp/document.pdf s3://my-docs-bucket/docs/doc-001.pdf
```

Then insert the reference:

```sql
INSERT INTO documents VALUES ('doc-001', 'Report Q1', 'docs/doc-001.pdf', 2048000, 'application/pdf', now());
```

## Strategy 3: Use the CODEC for Compression

Apply aggressive compression to large string columns:

```sql
CREATE TABLE logs (
    timestamp DateTime,
    host String,
    message String CODEC(ZSTD(3))
) ENGINE = MergeTree()
ORDER BY (host, timestamp);
```

For semi-structured JSON blobs, ZSTD often achieves 5-10x compression.

## Strategy 4: Column-Level TTL to Expire Blobs

Keep blobs only for a limited time:

```sql
ALTER TABLE events
    MODIFY COLUMN raw_payload String TTL timestamp + INTERVAL 7 DAY;
```

After 7 days, `raw_payload` is set to empty string while other columns are retained.

## Strategy 5: Separate Hot and Cold Tables

Keep large blobs in a separate, less-queried table:

```sql
-- Hot table: frequently queried metadata
CREATE TABLE events_meta (
    event_id String,
    event_type String,
    timestamp DateTime
) ENGINE = MergeTree()
ORDER BY (event_type, timestamp);

-- Cold table: large payloads accessed rarely
CREATE TABLE events_payload (
    event_id String,
    raw_payload String CODEC(ZSTD(9))
) ENGINE = MergeTree()
ORDER BY event_id;
```

## Summary

ClickHouse is not designed for blob storage. Handling large fields requires truncation, external object storage for binary data, aggressive compression codecs, column TTLs, and separating metadata from payload tables. These strategies keep your analytical queries fast while still making blobs accessible when needed.
