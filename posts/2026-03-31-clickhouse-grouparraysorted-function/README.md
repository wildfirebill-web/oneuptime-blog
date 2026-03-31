# How to Use groupArraySorted() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, groupArraySorted

Description: Collect the top-N values in sorted order from a group using groupArraySorted() in ClickHouse, with better performance than a full sort followed by arraySlice.

---

Collecting a sorted list of the top N values within each group is a common reporting need - recent events per user, top products per category, highest-latency requests per endpoint. ClickHouse's `groupArraySorted(N)(value)` does this in a single aggregation pass using an internal heap, avoiding the cost of sorting all values and then slicing.

## Syntax

```sql
groupArraySorted(N)(value)
-- or with explicit sort order:
groupArraySorted(N)(value, comparator)
```

- `N` - maximum number of elements to keep.
- `value` - the expression to collect.
- `comparator` (optional) - `'asc'` (default) or `'desc'`.

Returns `Array(T)` containing up to N elements in ascending order (or descending if specified).

## Basic Example

```sql
SELECT groupArraySorted(3)(number) AS top3
FROM numbers(10);
-- Returns [0, 1, 2]

SELECT groupArraySorted(3)('desc')(number) AS top3_desc
FROM numbers(10);
-- Returns [9, 8, 7]
```

## Practical Table: Top Latencies Per Endpoint

```sql
CREATE TABLE api_requests
(
    endpoint   String,
    latency_ms UInt32,
    ts         DateTime
)
ENGINE = MergeTree()
ORDER BY (endpoint, ts);

INSERT INTO api_requests VALUES
    ('/api/users',   120, '2024-01-01 10:00:00'),
    ('/api/users',   450, '2024-01-01 10:01:00'),
    ('/api/users',    80, '2024-01-01 10:02:00'),
    ('/api/users',   300, '2024-01-01 10:03:00'),
    ('/api/orders',  200, '2024-01-01 10:00:00'),
    ('/api/orders', 1200, '2024-01-01 10:01:00'),
    ('/api/orders',  350, '2024-01-01 10:02:00');
```

```sql
SELECT
    endpoint,
    groupArraySorted(3)('desc')(latency_ms) AS top3_latencies
FROM api_requests
GROUP BY endpoint
ORDER BY endpoint;
```

```text
endpoint      | top3_latencies
--------------|---------------
/api/orders   | [1200, 350, 200]
/api/users    | [450, 300, 120]
```

## Collecting Recent Events Per User

Use `groupArraySorted` with a timestamp to get the most recent N events per user:

```sql
CREATE TABLE user_activity
(
    user_id    UInt32,
    action     String,
    ts         DateTime
)
ENGINE = MergeTree()
ORDER BY (user_id, ts);

INSERT INTO user_activity VALUES
    (1, 'login',    '2024-01-01 08:00:00'),
    (1, 'view',     '2024-01-01 09:00:00'),
    (1, 'purchase', '2024-01-01 10:00:00'),
    (1, 'logout',   '2024-01-01 11:00:00'),
    (2, 'login',    '2024-01-01 08:30:00'),
    (2, 'view',     '2024-01-01 08:45:00');
```

```sql
SELECT
    user_id,
    groupArraySorted(3)('desc')(ts)    AS recent_timestamps,
    groupArraySorted(3)('desc')(action, ts) AS recent_actions
FROM user_activity
GROUP BY user_id
ORDER BY user_id;
```

When sorting by a secondary column (e.g., sort by `ts` to determine which `action` values to keep), pass a tuple:

```sql
SELECT
    user_id,
    arrayMap(t -> t.2, groupArraySorted(3)('desc')((ts, action))) AS recent_actions
FROM user_activity
GROUP BY user_id
ORDER BY user_id;
```

```text
user_id | recent_actions
--------|-----------------
1       | ['logout','purchase','view']
2       | ['view','login']
```

## Performance: groupArraySorted vs ORDER BY + LIMIT

For large groups, `groupArraySorted(N)` is significantly faster because it maintains only a heap of size N rather than sorting all rows:

```sql
-- Slower: sorts all rows then slices
SELECT
    endpoint,
    arraySlice(arraySort('x -> -x', groupArray(latency_ms)), 1, 3) AS top3
FROM api_requests
GROUP BY endpoint;

-- Faster: heap-based, keeps only top N
SELECT
    endpoint,
    groupArraySorted(3)('desc')(latency_ms) AS top3
FROM api_requests
GROUP BY endpoint;
```

The heap approach is O(n log N) versus O(n log n) for a full sort - a meaningful difference when groups have millions of rows.

## Combine with arrayJoin to Flatten Results

```sql
SELECT
    endpoint,
    rank,
    latency_ms
FROM (
    SELECT
        endpoint,
        groupArraySorted(5)('desc')(latency_ms) AS top5
    FROM api_requests
    GROUP BY endpoint
)
ARRAY JOIN top5 AS latency_ms, arrayEnumerate(top5) AS rank
ORDER BY endpoint, rank;
```

## Summary

`groupArraySorted(N)(value)` collects the top N values per group in sorted order using an internal heap, making it faster than collecting all values with `groupArray` and then sorting. Pass `'desc'` as the comparator to get the largest N values, or use tuple expressions to sort by one column while collecting another. It is the idiomatic ClickHouse approach to per-group top-N reporting.
