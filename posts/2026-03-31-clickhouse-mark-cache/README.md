# How to Configure ClickHouse Mark Cache for Better Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Database, Performance, Caching, Configuration

Description: Learn how to size and tune the ClickHouse mark cache to reduce I/O on index mark files and accelerate query execution across MergeTree tables.

---

ClickHouse MergeTree tables store data in sorted order and use a sparse primary index to locate data. Each column also has mark files (`.mrk` or `.mrk2`) that record the byte offset of every granule in the compressed column data files. Every query that reads a column must first load the relevant mark files to know where to seek. The mark cache keeps these mark files in RAM so they do not need to be re-read from disk on every query.

## What Mark Files Contain

For a MergeTree table, each part directory contains one mark file per column. A mark file holds one entry per granule (default 8192 rows) with the compressed file offset and the uncompressed offset within that block. On a table with billions of rows and dozens of columns, mark files can collectively occupy hundreds of megabytes. Loading them from disk on every query creates measurable latency.

## Configuring the Mark Cache Size

Set `mark_cache_size` in `/etc/clickhouse-server/config.xml` or a drop-in file in `/etc/clickhouse-server/config.d/`:

```xml
<clickhouse>
    <!-- Mark cache size in bytes. 5 GiB shown here. -->
    <mark_cache_size>5368709120</mark_cache_size>
</clickhouse>
```

Reference values in bytes:

```text
512 MiB =  536870912
1 GiB   = 1073741824
2 GiB   = 2147483648
5 GiB   = 5368709120
10 GiB  = 10737418240
```

The default is 5 GiB. For servers with many large tables, increasing this value reduces the chance that frequently used mark files get evicted.

## Estimating the Size of Your Mark Files

You can query ClickHouse directly to find the total mark file size:

```sql
-- Total size of all .mrk / .mrk2 files on this server
SELECT
    formatReadableSize(sum(marks_bytes)) AS total_marks_size,
    count() AS part_count
FROM system.parts
WHERE active;
```

```sql
-- Per-table breakdown
SELECT
    database,
    table,
    formatReadableSize(sum(marks_bytes)) AS marks_size,
    sum(rows) AS total_rows
FROM system.parts
WHERE active
GROUP BY database, table
ORDER BY sum(marks_bytes) DESC
LIMIT 20;
```

If your total marks size is 3 GiB and the cache is set to 5 GiB, your entire mark index fits in cache. If marks exceed the cache size you will see evictions and repeated disk reads.

## Monitoring Cache Effectiveness

```sql
-- Cumulative hit / miss counts since server start
SELECT
    event,
    value
FROM system.events
WHERE event IN (
    'MarkCacheHits',
    'MarkCacheMisses'
)
ORDER BY event;
```

Calculate the hit rate:

```sql
SELECT
    (SELECT value FROM system.events WHERE event = 'MarkCacheHits') AS hits,
    (SELECT value FROM system.events WHERE event = 'MarkCacheMisses') AS misses,
    round(hits / (hits + misses) * 100, 2) AS hit_rate_pct;
```

A well-tuned system should see a hit rate above 90% for steady-state workloads. A rate below 70% typically means the cache is too small relative to the working set of active parts.

## Flushing the Mark Cache

```sql
SYSTEM DROP MARK CACHE;
```

Use this during benchmarking to get cold-cache measurements, or after a large data migration when you want the cache to re-populate with fresh marks.

## Interaction with the Index and Uncompressed Cache

ClickHouse has three layered caches for MergeTree reads:

1. **Primary index** - loaded into RAM at table attach time, not controlled by mark_cache_size.
2. **Mark cache** - caches `.mrk` files so ClickHouse knows block offsets.
3. **Uncompressed cache** - caches already-decompressed column data blocks.

Tuning all three together gives the best results:

```xml
<clickhouse>
    <!-- 5 GiB for marks -->
    <mark_cache_size>5368709120</mark_cache_size>

    <!-- 16 GiB for decompressed blocks -->
    <uncompressed_cache_size>17179869184</uncompressed_cache_size>
</clickhouse>
```

## Effect of index_granularity on Mark File Size

The `index_granularity` table setting (default 8192) controls how many rows each mark covers. Smaller granularity means more marks, larger mark files, and more cache pressure but finer data skipping. Larger granularity reduces mark file size at the cost of reading slightly more data per granule.

```sql
-- Check index_granularity for existing tables
SELECT
    database,
    name AS table,
    engine_full
FROM system.tables
WHERE engine LIKE 'MergeTree%'
  AND database NOT IN ('system', 'information_schema')
ORDER BY database, table;
```

When you have tables with very small granularity (e.g., 1024), mark files grow proportionally and you may need a larger mark cache.

## Adaptive Granularity and Mark Cache

If `index_granularity_bytes` is set (default 10 MiB), ClickHouse uses adaptive granularity and writes `.mrk2` files. These are slightly larger than fixed-granularity `.mrk` files but reduce I/O by ensuring compressed blocks are not too large. The mark cache handles both formats transparently.

```sql
-- Check whether adaptive granularity is active for a table
SELECT
    database,
    name,
    create_table_query
FROM system.tables
WHERE database = 'my_db'
  AND name = 'my_table';
```

## Practical Sizing Guidelines

| Total Active Marks Size | Recommended mark_cache_size |
|---|---|
| Under 1 GiB | 2 GiB (default is fine) |
| 1-4 GiB | 5-8 GiB |
| 4-10 GiB | 12-16 GiB |
| Over 10 GiB | Equals or exceeds total marks size |

When in doubt, run the marks size query above, double the result, and use that as your cache size. The mark files are much smaller than the raw data, so increasing the mark cache rarely costs more than a few gigabytes.

## Conclusion

The mark cache is a low-cost, high-value configuration option for ClickHouse. Set `mark_cache_size` to at least the total size of your active mark files, monitor hit rates via `system.events`, and pair it with appropriate `uncompressed_cache_size` settings. A properly warmed mark cache eliminates repeated disk I/O for index lookups and noticeably improves query latency for selective range scans.
