# How to Use MinMax Skip Index in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Index, MinMax, Database, Performance, Query

Description: Learn how to use the MinMax skip index in ClickHouse to accelerate range queries on numeric and date columns that are not part of the primary key.

---

The MinMax skip index is the simplest and most efficient skip index in ClickHouse. It stores the minimum and maximum values for a column within each skip index block (a group of granules). When a query includes a range filter, ClickHouse checks each block's min/max values and skips any block where the filter range cannot overlap with [min, max].

## How MinMax Works

For a column with values distributed across the table:

```text
Block 0: min=100, max=200
Block 1: min=201, max=350
Block 2: min=350, max=500
```

A query `WHERE val > 400` skips blocks 0 and 1, reading only block 2.

MinMax is exact for the blocks it can skip: if the filter range has no overlap with [min, max], the block is guaranteed to contain no matches.

## When MinMax Is Effective

MinMax works best when:

- Column values are correlated with the sort order of the table
- Queries use range filters (`>`, `<`, `BETWEEN`, `>=`, `<=`)
- The column has moderate-to-high clustering within granules

It is ineffective when values are uniformly distributed across all granules, because every block's [min, max] range overlaps with any filter.

## Creating a MinMax Index

```sql
CREATE TABLE server_metrics
(
    server_id   UInt32,
    cpu_pct     Float32,
    mem_used_mb UInt32,
    disk_io_mb  Float32,
    ts          DateTime,

    INDEX idx_cpu    cpu_pct     TYPE minmax GRANULARITY 4,
    INDEX idx_mem    mem_used_mb TYPE minmax GRANULARITY 4,
    INDEX idx_diskio disk_io_mb  TYPE minmax GRANULARITY 4
)
ENGINE = MergeTree()
ORDER BY (server_id, ts);
```

## Query That Benefits from MinMax

```sql
SELECT
    server_id,
    avg(cpu_pct)
FROM server_metrics
WHERE cpu_pct > 90
  AND ts > now() - INTERVAL 1 DAY
GROUP BY server_id
ORDER BY avg(cpu_pct) DESC;
```

Without the index, ClickHouse reads every granule. With the MinMax index on `cpu_pct`, it skips all blocks where `max(cpu_pct) <= 90`.

## Verifying with EXPLAIN

```sql
EXPLAIN indexes = 1
SELECT count()
FROM server_metrics
WHERE cpu_pct > 90;
```

Expected output includes:

```text
Skip
  Name: idx_cpu
  Description: minmax GRANULARITY 4
  Granules: 8/500
```

This shows 492 granules were skipped.

## Adding MinMax to an Existing Table

```sql
ALTER TABLE server_metrics
    ADD INDEX idx_ts_range ts TYPE minmax GRANULARITY 8;

ALTER TABLE server_metrics MATERIALIZE INDEX idx_ts_range;
```

## Choosing GRANULARITY

The `GRANULARITY` parameter determines how many primary index granules are combined into one skip index entry:

```sql
-- Fine-grained: more index entries, better selectivity, larger index file
INDEX idx_cpu cpu_pct TYPE minmax GRANULARITY 1

-- Coarse-grained: fewer entries, less memory, potentially less selectivity
INDEX idx_cpu cpu_pct TYPE minmax GRANULARITY 16
```

For a table with 8192 rows per granule:

- `GRANULARITY 1` = one MinMax entry per 8192 rows
- `GRANULARITY 4` = one MinMax entry per 32768 rows
- `GRANULARITY 8` = one MinMax entry per 65536 rows

Start with `GRANULARITY 4`. Decrease to 1 if queries are highly selective; increase to 8-16 if the index consumes too much memory.

## MinMax on DateTime Columns

DateTime columns outside the primary key benefit greatly from MinMax when queries filter on time ranges:

```sql
CREATE TABLE audit_events
(
    tenant_id  UInt32,
    event_type String,
    user_id    UInt64,
    ts         DateTime,

    INDEX idx_ts ts TYPE minmax GRANULARITY 4
)
ENGINE = MergeTree()
ORDER BY (tenant_id, event_type);
```

Query:

```sql
SELECT count()
FROM audit_events
WHERE ts > '2024-06-01'
  AND ts < '2024-06-30';
```

## MinMax on Integer Columns

```sql
CREATE TABLE orders
(
    order_id    UInt64,
    customer_id UInt32,
    total_cents UInt64,
    ts          DateTime,

    INDEX idx_total total_cents TYPE minmax GRANULARITY 4
)
ENGINE = MergeTree()
ORDER BY (ts, order_id);
```

Query for large orders:

```sql
SELECT order_id, total_cents
FROM orders
WHERE total_cents > 100000
ORDER BY total_cents DESC
LIMIT 100;
```

## Checking Index Memory Usage

```sql
SELECT
    name,
    type,
    formatReadableSize(data_compressed_bytes) AS compressed_size
FROM system.data_skipping_indices
WHERE table = 'server_metrics'
  AND database = currentDatabase();
```

## MinMax vs Other Skip Index Types

| Use case | Index type |
|----------|-----------|
| Range filter on numeric column | minmax |
| Equality on low-cardinality column | set(N) |
| Equality on high-cardinality column | bloom_filter |
| LIKE substring matching | ngrambf_v1 |
| Full-text token matching | tokenbf_v1 |

## Summary

MinMax is the lowest-overhead skip index in ClickHouse. It costs almost no storage (just two values per block) and provides exact range pruning. Use it on any numeric or datetime column that appears in range filter conditions but is not part of the primary key. Always confirm effectiveness with `EXPLAIN indexes = 1`.
