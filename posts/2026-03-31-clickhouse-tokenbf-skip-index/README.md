# How to Use tokenbf_v1 Skip Index for Text Search in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Skip Index, tokenbf_v1, Text Search, Full-Text Search

Description: Learn how to use the tokenbf_v1 skip index in ClickHouse for fast word-token-based text search, ideal for log messages and structured text fields.

---

## What is tokenbf_v1

`tokenbf_v1` is a skip index that tokenizes strings by non-alphanumeric separators and stores a bloom filter of the resulting tokens per granule. Unlike `ngrambf_v1`, it works at the word level - ideal for natural language text, log messages, and URL paths where you search for whole words or tokens.

## Create a Table with tokenbf_v1

```sql
CREATE TABLE logs (
    ts DateTime,
    level LowCardinality(String),
    message String,
    INDEX idx_tokens message TYPE tokenbf_v1(65536, 3, 0) GRANULARITY 1
) ENGINE = MergeTree()
ORDER BY ts;
```

Parameters: `tokenbf_v1(bloom_filter_size, hash_functions, seed)`

- `bloom_filter_size = 65536` - bytes per bloom filter
- `hash_functions = 3` - number of hash functions
- `seed = 0` - random seed

## Insert Sample Log Data

```sql
INSERT INTO logs VALUES
    ('2026-01-01 10:00:00', 'ERROR', 'Connection refused to database server'),
    ('2026-01-01 10:01:00', 'INFO', 'User authentication successful'),
    ('2026-01-01 10:02:00', 'ERROR', 'Timeout connecting to upstream service'),
    ('2026-01-01 10:03:00', 'WARN', 'High memory usage detected on node-01');
```

## Query with hasToken

```sql
SELECT ts, level, message
FROM logs
WHERE hasToken(message, 'Connection');
```

```sql
SELECT ts, level, message
FROM logs
WHERE hasToken(message, 'Timeout');
```

## Query with LIKE for Full Words

```sql
SELECT ts, level, message
FROM logs
WHERE message LIKE '%database%';
```

`tokenbf_v1` can also help with `LIKE` queries when the pattern matches whole tokens.

## Verify Index Usage

```sql
EXPLAIN indexes = 1
SELECT ts, message
FROM logs
WHERE hasToken(message, 'Connection');
```

## tokenbf_v1 vs ngrambf_v1

```text
tokenbf_v1:
- Works on whole words/tokens
- Better for natural language and log searches
- Useless for substring matches within words
- Smaller index for typical log data

ngrambf_v1:
- Works on any substring
- Better for partial matches (URL subpaths, partial words)
- Larger index due to more n-gram entries
- Higher false positive rate for short substrings
```

## Multiple Indexes on One Column

```sql
CREATE TABLE logs_dual (
    ts DateTime,
    message String,
    INDEX idx_token message TYPE tokenbf_v1(32768, 3, 0) GRANULARITY 1,
    INDEX idx_ngram message TYPE ngrambf_v1(4, 32768, 2, 0) GRANULARITY 1
) ENGINE = MergeTree()
ORDER BY ts;
```

Using both lets token-based queries use `tokenbf_v1` and substring queries use `ngrambf_v1`.

## Bloom Filter Size Tuning

```sql
-- Small, higher false-positive rate
INDEX idx_small message TYPE tokenbf_v1(8192, 2, 0) GRANULARITY 1

-- Large, lower false-positive rate
INDEX idx_large message TYPE tokenbf_v1(131072, 5, 0) GRANULARITY 1
```

Check index disk usage:

```sql
SELECT
    formatReadableSize(sum(secondary_indices_bytes_on_disk)) AS index_size
FROM system.parts
WHERE table = 'logs' AND active = 1;
```

## Summary

The `tokenbf_v1` skip index in ClickHouse enables fast token-based text search by storing word bloom filters per granule. Use it with `hasToken()` for exact word matches in log messages, event descriptions, and structured text. For partial substring searches, combine with or prefer `ngrambf_v1`.
