# ClickHouse Skip Index Types Feature Comparison

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Skip Index, Bloom Filter, Performance, Query Optimization

Description: A comparison of ClickHouse data skipping index types - minmax, set, bloom_filter, ngrambf_v1, and tokenbf_v1 - with use cases and SQL examples.

---

## What Are Skip Indexes

ClickHouse stores data in granules (default 8192 rows). Skip indexes store metadata per granule that allows ClickHouse to skip granules that cannot match a WHERE clause, reducing I/O without reading the actual column data.

## Index Types Overview

```text
Index Type   | Column Types         | Query Types           | Index Size
-------------|----------------------|-----------------------|------------
minmax       | Numeric, Date        | Range (>, <, BETWEEN) | Tiny
set          | Any low-cardinality  | Equality (=, IN)      | Small
bloom_filter | Any                  | Equality, IN          | Medium
ngrambf_v1   | String               | LIKE '%substr%'       | Larger
tokenbf_v1   | String               | Full-word token match | Medium
```

## minmax Index

Stores the minimum and maximum value per granule. Best for monotonically increasing or range-queried numeric columns.

```sql
ALTER TABLE events
ADD INDEX idx_amount amount TYPE minmax GRANULARITY 4;

-- Benefits this query
SELECT * FROM events WHERE amount BETWEEN 100 AND 500;
```

## set Index

Stores the set of distinct values per granule. Useful for low-cardinality columns queried with `=` or `IN`.

```sql
ALTER TABLE events
ADD INDEX idx_status status TYPE set(100) GRANULARITY 1;

-- Benefits this query
SELECT * FROM events WHERE status = 'completed';
```

The `set(100)` parameter is the maximum distinct values to track per granule. Granules with more unique values are not indexed.

## bloom_filter Index

Probabilistic filter - can produce false positives but never false negatives. Good for high-cardinality equality searches.

```sql
ALTER TABLE events
ADD INDEX idx_session_id session_id TYPE bloom_filter(0.01) GRANULARITY 4;

-- Benefits equality and IN queries
SELECT * FROM events WHERE session_id = 'abc123xyz';
SELECT * FROM events WHERE session_id IN ('abc123', 'def456');
```

The `0.01` is the false positive rate. Lower rate means larger index.

## ngrambf_v1 Index

Splits strings into n-grams and stores them in a Bloom filter. Best for substring searches.

```sql
ALTER TABLE logs
ADD INDEX idx_message message TYPE ngrambf_v1(4, 65536, 2, 0) GRANULARITY 4;

-- Benefits LIKE queries
SELECT * FROM logs WHERE message LIKE '%OutOfMemoryError%';
```

Parameters: ngram size, Bloom filter size (bytes), hash functions, seed.

## tokenbf_v1 Index

Tokenizes strings by non-alphanumeric characters and stores tokens in a Bloom filter. Better than ngrambf for full-word search.

```sql
ALTER TABLE logs
ADD INDEX idx_tokens message TYPE tokenbf_v1(65536, 2, 0) GRANULARITY 4;

-- Benefits full-word token queries
SELECT * FROM logs WHERE hasToken(message, 'error');
```

## Verifying Skip Index Usage

```sql
EXPLAIN indexes = 1
SELECT * FROM events WHERE session_id = 'abc123';
```

Look for `Granules: 5/1000` meaning 995 granules were skipped.

## Summary

Choose `minmax` for range queries on numeric or date columns, `set` for low-cardinality equality filters, `bloom_filter` for high-cardinality equality and IN conditions, `ngrambf_v1` for substring LIKE searches, and `tokenbf_v1` for full-word token matching. Always verify with `EXPLAIN indexes = 1` that an index actually skips granules before keeping it in production.
