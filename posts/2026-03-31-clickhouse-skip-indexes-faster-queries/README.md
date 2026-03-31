# How to Use Skip Indexes in ClickHouse for Faster Queries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Index, SkipIndex, Database, Performance, Query

Description: Learn how ClickHouse data-skipping indexes work, how to create and configure them, and when to use each type for faster analytical queries.

---

ClickHouse primary indexes only help when queries filter on the leading columns of the `ORDER BY` key. For all other columns, ClickHouse reads every granule. Data-skipping indexes (also called skip indexes or secondary indexes) solve this by storing per-granule metadata that allows ClickHouse to skip granules that cannot contain matching rows, without physically sorting data by those columns.

## How Skip Indexes Work

Skip indexes store aggregated metadata about each granule (or group of granules) for a specific column. Before reading a granule, ClickHouse checks the index. If the metadata proves the granule cannot satisfy the query's `WHERE` condition, the granule is skipped entirely.

Unlike B-tree indexes, skip indexes are approximate. They can produce false positives (read a granule that turns out to have no matches) but never false negatives (skip a granule that has matches).

## Types of Skip Indexes

| Index type | Stores | Best for |
|------------|--------|----------|
| `minmax` | Min and max value per granule | Range filters on numeric/date columns |
| `set(n)` | Up to n distinct values per granule | Low-cardinality equality filters |
| `bloom_filter` | Probabilistic membership | High-cardinality equality and IN filters |
| `ngrambf_v1` | n-gram bloom filter | LIKE pattern matching on strings |
| `tokenbf_v1` | Token-based bloom filter | Full-text token matching |

## Creating a Skip Index

Skip indexes are defined in the `CREATE TABLE` statement using the `INDEX` clause, or added later with `ALTER TABLE`:

```sql
CREATE TABLE http_logs
(
    domain     String,
    status     UInt16,
    method     String,
    url        String,
    bytes      UInt32,
    ts         DateTime,

    INDEX idx_status   status   TYPE minmax      GRANULARITY 4,
    INDEX idx_method   method   TYPE set(10)     GRANULARITY 4,
    INDEX idx_url      url      TYPE bloom_filter GRANULARITY 4
)
ENGINE = MergeTree()
ORDER BY (domain, ts);
```

## The GRANULARITY Parameter

The `GRANULARITY` parameter controls how many primary index granules are merged into one skip index block. A value of 4 means each skip index entry covers 4 * 8192 = 32768 rows.

- **Lower granularity**: more precise (fewer false positives), larger index size
- **Higher granularity**: less precise, smaller index size

Start with `GRANULARITY 4` and adjust based on your data distribution.

## Adding a Skip Index to an Existing Table

```sql
ALTER TABLE http_logs
    ADD INDEX idx_bytes bytes TYPE minmax GRANULARITY 4;
```

After adding the index, it only applies to new parts. To build the index on existing data:

```sql
ALTER TABLE http_logs MATERIALIZE INDEX idx_bytes;
```

Or force a merge:

```sql
OPTIMIZE TABLE http_logs FINAL;
```

## Verifying Skip Index Usage

Use `EXPLAIN indexes = 1` to see whether a skip index is being consulted:

```sql
EXPLAIN indexes = 1
SELECT count()
FROM http_logs
WHERE status = 404;
```

The output includes lines like:

```text
ReadFromMergeTree
  Indexes:
    PrimaryKey
      Condition: ...
      Granules: 100/100
    Skip
      Name: idx_status
      Description: minmax
      Granules: 3/100
```

This shows the primary key read all 100 granules, but the minmax skip index narrowed it down to 3.

## Measuring Skip Index Effectiveness

```sql
SELECT
    table,
    name,
    type,
    formatReadableSize(data_compressed_bytes) AS index_size
FROM system.data_skipping_indices
WHERE table = 'http_logs'
  AND database = currentDatabase();
```

Also query `system.query_log` after running queries:

```sql
SELECT
    query,
    read_rows,
    read_bytes,
    query_duration_ms
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query LIKE '%http_logs%'
ORDER BY event_time DESC
LIMIT 10;
```

Compare `read_rows` before and after adding skip indexes.

## Dropping a Skip Index

```sql
ALTER TABLE http_logs
    DROP INDEX idx_bytes;
```

## Skip Index Limitations

Skip indexes are not silver bullets:

- They only skip at granule granularity. If matching rows are scattered across all granules, the index provides no benefit.
- High-cardinality columns with uniformly distributed values benefit little from skip indexes.
- Set indexes with too few allowed values (`set(10)`) overflow and become useless on high-cardinality columns.
- Bloom filter indexes have a configurable false-positive rate. Tune `bloom_filter(false_positive_rate)` for your workload.

## Choosing the Right Skip Index

```sql
-- For range queries on numeric columns
INDEX idx_bytes bytes TYPE minmax GRANULARITY 4

-- For equality queries on a low-cardinality column
INDEX idx_status status TYPE set(100) GRANULARITY 4

-- For equality and IN queries on high-cardinality strings
INDEX idx_user_id user_id TYPE bloom_filter(0.01) GRANULARITY 4

-- For LIKE pattern matching
INDEX idx_message message TYPE ngrambf_v1(4, 65536, 2, 0) GRANULARITY 4

-- For token-based text search
INDEX idx_message_token message TYPE tokenbf_v1(65536, 2, 0) GRANULARITY 4
```

## Summary

Skip indexes extend ClickHouse's pruning capabilities beyond the primary index. Use `minmax` for numeric range filters, `set` for low-cardinality equality filters, `bloom_filter` for high-cardinality equality filters, and ngram/token bloom filters for string pattern matching. Always verify effectiveness with `EXPLAIN indexes = 1` and measure `read_rows` reduction in `system.query_log`.
