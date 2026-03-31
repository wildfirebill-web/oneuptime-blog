# How to Use Map Data Type in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Data Types, Map

Description: Learn how to store key-value pairs in ClickHouse using Map(K, V), access values by key, and use mapKeys() and mapValues() functions.

---

`Map(K, V)` stores an associative array of key-value pairs within a single column. The key type `K` must be a non-nullable type (typically `String` or an integer), and the value type `V` can be any ClickHouse type. Maps are useful for dynamic attributes, HTTP headers, labels, configuration blobs, and other scenarios where keys are not known at schema design time.

## Defining Map Columns

Use `Map(K, V)` in the column definition. Map literals are written with curly-brace syntax: `{'key': value, ...}`.

```sql
CREATE TABLE http_logs (
    request_id   UInt64,
    url          String,
    headers      Map(String, String),
    query_params Map(String, String),
    timing_ms    Map(String, UInt32),
    logged_at    DateTime
) ENGINE = MergeTree()
ORDER BY (request_id, logged_at);

INSERT INTO http_logs VALUES
(
    1,
    '/api/v1/users',
    {'Content-Type': 'application/json', 'Authorization': 'Bearer abc123'},
    {'page': '1', 'limit': '50'},
    {'dns': 2, 'connect': 15, 'ttfb': 42},
    '2026-03-01 10:00:00'
),
(
    2,
    '/api/v1/orders',
    {'Content-Type': 'application/json'},
    {'status': 'pending'},
    {'dns': 1, 'connect': 12, 'ttfb': 38},
    '2026-03-01 10:01:00'
);
```

## Accessing Values by Key

Use bracket notation `map[key]` to retrieve a value. Returns the default value for the type if the key is absent (empty string for String, 0 for numbers).

```sql
-- Access a specific key
SELECT
    request_id,
    headers['Content-Type']    AS content_type,
    query_params['page']       AS page,
    timing_ms['ttfb']          AS ttfb_ms
FROM http_logs;

-- Check key existence with mapContains
SELECT request_id
FROM http_logs
WHERE mapContains(headers, 'Authorization');
```

## mapKeys and mapValues

Extract all keys or all values from a map as an `Array`.

```sql
-- Get all keys present in headers
SELECT request_id, mapKeys(headers) AS header_names FROM http_logs;

-- Get all values from timing_ms
SELECT request_id, mapValues(timing_ms) AS timing_values FROM http_logs;

-- Sum all timing values for each request
SELECT
    request_id,
    arrayReduce('sum', mapValues(timing_ms)) AS total_ms
FROM http_logs;
```

## Querying and Filtering Maps

Combine map access with standard WHERE and aggregation clauses.

```sql
-- Find requests taking more than 40ms for TTFB
SELECT request_id, url, timing_ms['ttfb'] AS ttfb
FROM http_logs
WHERE timing_ms['ttfb'] > 40;

-- Count requests by Content-Type header
SELECT
    headers['Content-Type'] AS content_type,
    count() AS request_count
FROM http_logs
GROUP BY content_type
ORDER BY request_count DESC;

-- Expand map into rows using ARRAY JOIN on mapKeys/mapValues
SELECT
    request_id,
    key,
    timing_ms[key] AS duration_ms
FROM http_logs
ARRAY JOIN mapKeys(timing_ms) AS key;
```

## Map vs Nested - When to Use Each

Maps and Nested types both represent structured sub-data but have different trade-offs.

```sql
-- Map: best when keys are dynamic and not known ahead of time
-- Nested: best when the set of sub-columns is fixed and you want
--         per-column compression and filtering

-- Example: use Map for arbitrary user labels
CREATE TABLE resources (
    resource_id UInt64,
    labels      Map(String, String),   -- dynamic k/v labels
    created_at  DateTime
) ENGINE = MergeTree()
ORDER BY resource_id;

INSERT INTO resources VALUES
    (1, {'env': 'prod', 'team': 'platform', 'tier': 'critical'}, '2026-01-01 00:00:00'),
    (2, {'env': 'staging', 'team': 'backend'}, '2026-01-02 00:00:00');

-- Find all production resources
SELECT resource_id FROM resources WHERE labels['env'] = 'prod';
```

## Modifying Maps

Maps in ClickHouse are immutable per row but can be constructed or merged using functions.

```sql
-- mapUpdate: merge two maps (second map overwrites duplicate keys)
SELECT mapUpdate({'a': 1, 'b': 2}, {'b': 99, 'c': 3}) AS merged;

-- Build a map from arrays using mapFromArrays
SELECT mapFromArrays(['dns', 'connect', 'ttfb'], [2, 15, 42]) AS timing;
```

## Summary

`Map(K, V)` in ClickHouse is the right choice for storing dynamic key-value attributes where keys are not fixed at schema design time. Use bracket access `map[key]` to retrieve values, `mapContains` to check for key presence, and `mapKeys`/`mapValues` to extract the full key and value sets as arrays. For fixed sub-schemas with known columns, prefer `Nested` or individual top-level columns for better compression and filter performance.
