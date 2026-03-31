# How to Translate Presto/Trino SQL to ClickHouse SQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Presto, Trino, Migration, Query Translation, Analytics

Description: Learn how to convert Presto and Trino SQL queries to ClickHouse SQL, including function mappings, type systems, and structural query differences.

---

## Why Move from Presto/Trino to ClickHouse

Presto and Trino are federated query engines that read data from multiple sources. They are flexible but query latency depends heavily on the underlying storage. ClickHouse stores data in its own columnar format and typically delivers faster single-table analytics. Teams often migrate hot analytical queries to ClickHouse while keeping Trino for cross-system joins.

## Data Type Mapping

```sql
-- Presto/Trino        ClickHouse
INTEGER             -> Int32
BIGINT              -> Int64
DOUBLE              -> Float64
DECIMAL(p,s)        -> Decimal(p,s)
VARCHAR(n)          -> String
BOOLEAN             -> UInt8
TIMESTAMP           -> DateTime64(3)
DATE                -> Date
ARRAY(T)            -> Array(T)
MAP(K,V)            -> Map(K,V)
ROW(...)            -> Tuple(...)
```

## Date and Time Functions

```sql
-- Presto/Trino
SELECT
  date_trunc('month', created_at) AS month,
  date_add('day', 7, created_at) AS week_later,
  date_diff('second', start_time, end_time) AS duration_s
FROM sessions;

-- ClickHouse
SELECT
  toStartOfMonth(created_at) AS month,
  addDays(created_at, 7) AS week_later,
  dateDiff('second', start_time, end_time) AS duration_s
FROM sessions;
```

## ARRAY Functions

```sql
-- Presto/Trino
SELECT cardinality(tags), array_join(tags, ','), contains(tags, 'promo')
FROM campaigns;

-- ClickHouse
SELECT length(tags), arrayStringConcat(tags, ','), has(tags, 'promo')
FROM campaigns;
```

## Unnesting Arrays

```sql
-- Presto/Trino
SELECT user_id, tag
FROM users CROSS JOIN UNNEST(tags) AS t(tag);

-- ClickHouse
SELECT user_id, arrayJoin(tags) AS tag
FROM users;
```

## String Functions

```sql
-- Presto/Trino
SELECT
  length(name),
  substr(name, 1, 5),
  strpos(name, 'admin'),
  regexp_extract(email, '@(.+)', 1)
FROM users;

-- ClickHouse
SELECT
  length(name),
  substring(name, 1, 5),
  position(name, 'admin'),
  extract(email, '@(.+)')
FROM users;
```

## APPROX_DISTINCT

```sql
-- Presto/Trino
SELECT approx_distinct(user_id) FROM events;

-- ClickHouse
SELECT uniq(user_id) FROM events;
```

## NULLIF

Both support `NULLIF` with the same syntax - no translation needed.

```sql
SELECT NULLIF(status, 'unknown') FROM records;
```

## Summary

Translating Presto/Trino to ClickHouse involves replacing `date_trunc` with `toStartOf*`, `date_add` with `addDays`/`addHours`, `cardinality` with `length`, `array_join` with `arrayStringConcat`, `contains` with `has`, and `CROSS JOIN UNNEST` with `arrayJoin`. Most string function names are close to standard SQL and translate with minor name changes. The `approx_distinct` function maps to `uniq`.
