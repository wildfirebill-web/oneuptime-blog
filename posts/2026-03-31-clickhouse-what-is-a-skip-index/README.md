# What Is a Skip Index and How It Works in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Skip Index, Secondary Index, MergeTree, Query Optimization

Description: Learn what skip indexes are in ClickHouse, how they work with granules to skip data during queries, and when to use each type.

---

Skip indexes (also called data-skipping indexes or secondary indexes) allow ClickHouse to skip granules that cannot contain matching rows, even when the query does not use the primary sort key. They are distinct from traditional B-tree indexes and work at the granule level.

## How Skip Indexes Work

A skip index stores summary information about each granule for a specific column. At query time, ClickHouse checks the index to determine whether a granule might contain rows matching the WHERE clause. If the index says no, the granule is skipped entirely.

```sql
ALTER TABLE events
ADD INDEX idx_region region TYPE set(100) GRANULARITY 1;
```

This creates a `set` index that stores up to 100 distinct values per granule. If the target value is not in the set, ClickHouse skips that granule.

## Skip Index Types

### minmax

Stores the minimum and maximum value of the column per granule. Useful for range queries on columns with monotonically increasing values.

```sql
ALTER TABLE events
ADD INDEX idx_revenue revenue TYPE minmax GRANULARITY 4;
```

```sql
-- Skips granules where min > 1000 or max < 500
SELECT * FROM events WHERE revenue BETWEEN 500 AND 1000;
```

### set

Stores up to N distinct values per granule. Useful for low-cardinality columns like `status`, `region`, or `event_type`.

```sql
ALTER TABLE events
ADD INDEX idx_event_type event_type TYPE set(50) GRANULARITY 1;
```

### bloom_filter

Uses a probabilistic Bloom filter per granule. Useful for high-cardinality string or UUID columns where you query for exact equality.

```sql
ALTER TABLE events
ADD INDEX idx_user_id user_id TYPE bloom_filter(0.01) GRANULARITY 1;
```

The `0.01` parameter is the false positive rate. Lower rates use more memory.

### tokenbf_v1 and ngrambf_v1

Token and n-gram Bloom filters for substring and full-text search patterns.

```sql
ALTER TABLE logs
ADD INDEX idx_message message TYPE tokenbf_v1(32768, 3, 0) GRANULARITY 1;
```

## GRANULARITY Parameter

The `GRANULARITY` parameter controls how many table granules are combined into one index granule. A value of 1 indexes every table granule. A value of 4 combines four table granules into one index granule, making the index smaller but less precise.

## When Skip Indexes Help

Skip indexes are most effective when:
- The query column is not the primary sort key
- The column has good clustering (similar values appear together)
- Queries frequently filter on this column

They provide no benefit when:
- Values are randomly distributed across granules (no skipping is possible)
- The column is already part of the primary sort key
- The table has very few parts or granules

## Verifying Skip Index Usage

```sql
-- Enable trace logging for a query
SET send_logs_level = 'trace';
SELECT count() FROM events WHERE region = 'US';
-- Look for "skip index" in the log output
```

Or check `system.query_log` for `read_rows` vs. `result_rows` to measure how many rows were skipped.

## Summary

Skip indexes work at the granule level, storing summaries (min/max, value sets, or Bloom filters) that let ClickHouse skip entire granules during a scan. Choose `minmax` for range queries on sorted columns, `set` for low-cardinality equality filters, and `bloom_filter` for high-cardinality equality lookups. Skip indexes complement the primary index but cannot replace it for the most selective queries.
