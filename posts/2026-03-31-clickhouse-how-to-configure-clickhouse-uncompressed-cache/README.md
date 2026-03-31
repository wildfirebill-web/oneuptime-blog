# How to Configure ClickHouse Uncompressed Cache

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Uncompressed Cache, Performance, Memory, Configuration

Description: Learn how to configure ClickHouse's uncompressed cache to store decompressed data blocks in memory, reducing CPU overhead for repeated queries on the same data.

---

## Overview

ClickHouse compresses column data on disk using codecs like LZ4 or ZSTD. Every time a query reads data, it must decompress the relevant blocks. The uncompressed cache stores already-decompressed data blocks in RAM so that repeated access to the same data blocks avoids redundant CPU decompression work.

## When to Use the Uncompressed Cache

The uncompressed cache is most beneficial when:
- The same data blocks are read repeatedly by many queries
- Your workload is CPU-bound due to decompression overhead
- Tables have high compression ratios (blocks are small compressed, large uncompressed)

It is less beneficial for:
- One-shot bulk scans of unique data
- Workloads limited by RAM rather than CPU

## Default Configuration

By default, the uncompressed cache is disabled (size = 0) in some ClickHouse versions, or set to a small value. Check current settings:

```sql
SELECT name, value FROM system.server_settings WHERE name = 'uncompressed_cache_size'
```

## Enabling and Sizing the Cache

Configure in `config.xml` or a file in `config.d/`:

```xml
<!-- /etc/clickhouse-server/config.d/uncompressed-cache.xml -->
<clickhouse>
    <uncompressed_cache_size>8589934592</uncompressed_cache_size>  <!-- 8 GiB -->
</clickhouse>
```

Restart the server to apply:

```bash
sudo systemctl restart clickhouse-server
```

## Per-Query Cache Usage Control

Even with the server-level cache enabled, you can control whether individual queries use it:

```sql
-- Enable uncompressed cache for this query
SET use_uncompressed_cache = 1;
SELECT avg(value) FROM metrics WHERE date = today();

-- Disable for a query that scans unique data (would pollute the cache)
SET use_uncompressed_cache = 0;
SELECT * FROM events WHERE date BETWEEN '2024-01-01' AND '2024-06-01';
```

## Monitoring Cache Performance

Check hit and miss rates:

```sql
SELECT
    event,
    value
FROM system.events
WHERE event IN ('UncompressedCacheHits', 'UncompressedCacheMisses', 'UncompressedCacheWeightLost')
```

Compute hit rate:

```sql
SELECT
    sumIf(value, event = 'UncompressedCacheHits')   AS hits,
    sumIf(value, event = 'UncompressedCacheMisses') AS misses,
    round(
        sumIf(value, event = 'UncompressedCacheHits') /
        nullIf(
            sumIf(value, event = 'UncompressedCacheHits') +
            sumIf(value, event = 'UncompressedCacheMisses'),
            0
        ) * 100,
        2
    ) AS hit_rate_pct
FROM system.events
WHERE event IN ('UncompressedCacheHits', 'UncompressedCacheMisses')
```

## Current Cache Size

```sql
SELECT metric, value, formatReadableSize(value) AS human_size
FROM system.metrics
WHERE metric = 'UncompressedCacheBytes'
```

## Clearing the Cache

```sql
SYSTEM DROP UNCOMPRESSED CACHE;
```

This is useful for benchmarking cold versus warm query performance.

## Sizing Recommendations

The uncompressed cache competes with the OS page cache and the mark cache for available RAM. A general guideline:

| Server RAM | Uncompressed Cache |
|------------|-------------------|
| 32 GB      | 4-8 GB            |
| 64 GB      | 8-16 GB           |
| 128 GB     | 16-32 GB          |

Reserve sufficient memory for ClickHouse query execution (controlled by `max_memory_usage`), the mark cache, and the OS page cache before sizing this cache.

## Trade-off vs. Page Cache

The OS page cache also caches compressed disk blocks. The uncompressed cache reduces CPU decompression time at the cost of storing larger (uncompressed) blocks in memory. If RAM is abundant and your queries are CPU-bound on decompression, the uncompressed cache helps. If RAM is scarce, relying on the page cache for compressed blocks may be more memory-efficient.

## Summary

ClickHouse's uncompressed cache stores decompressed data blocks in memory to eliminate redundant CPU decompression for repeated accesses. Configure `uncompressed_cache_size` in `config.xml`, enable per-query with `use_uncompressed_cache = 1`, and monitor `UncompressedCacheHits` vs `UncompressedCacheMisses` to validate effectiveness. Best for CPU-bound workloads with repeated access patterns.
