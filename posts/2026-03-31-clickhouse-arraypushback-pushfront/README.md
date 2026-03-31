# How to Use arrayPushBack() and arrayPushFront() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array Function, arrayPushBack, arrayPushFront

Description: Learn how arrayPushBack() and arrayPushFront() append and prepend elements to arrays in ClickHouse for building ordered lists, event logs, and incremental array construction.

---

ClickHouse arrays are immutable values - you cannot mutate them in place. However, you can construct new arrays by appending or prepending elements using `arrayPushBack` and `arrayPushFront`. These functions are the building blocks for incrementally assembling arrays in queries, building breadcrumb trails, maintaining ordered event logs, and constructing composite keys as arrays.

## Function Signatures

```text
arrayPushBack(arr, elem)  -> Array(T)
arrayPushFront(arr, elem) -> Array(T)
```

Both functions return a new array one element longer than the input. The element type must be compatible with the array's element type - ClickHouse will attempt implicit casting.

## Basic Appending and Prepending

```sql
-- Append to the end
SELECT arrayPushBack([1, 2, 3], 4) AS with_back;
-- Result: [1, 2, 3, 4]

-- Prepend to the front
SELECT arrayPushFront([1, 2, 3], 0) AS with_front;
-- Result: [0, 1, 2, 3]

-- Append a string
SELECT arrayPushBack(['hello', 'world'], 'clickhouse') AS str_arr;
-- Result: ['hello', 'world', 'clickhouse']

-- Push onto an empty array
SELECT arrayPushBack([], 42) AS from_empty;
-- Result: [42]
```

## Chaining Multiple Pushes

Since each function returns a new array, you can chain calls to build up an array incrementally in a single expression:

```sql
-- Build an array step by step
SELECT
    arrayPushBack(
        arrayPushBack(
            arrayPushBack([], 10),
            20
        ),
        30
    ) AS built_array;
-- Result: [10, 20, 30]

-- Add a header and footer to an existing array
SELECT arrayPushFront(arrayPushBack([2, 3, 4], 5), 1) AS framed;
-- Result: [1, 2, 3, 4, 5]
```

## Appending Event Logs with ALTER UPDATE

In MergeTree tables, `arrayPushBack` is frequently used inside `ALTER TABLE ... UPDATE` to append new events to an array column. Each mutation adds to the existing array:

```sql
CREATE TABLE user_sessions
(
    user_id UInt32,
    page_visits Array(String)
) ENGINE = MergeTree()
ORDER BY user_id;

INSERT INTO user_sessions VALUES
    (1, ['home', 'about']),
    (2, ['landing']);

-- Append a new page visit for user 1
ALTER TABLE user_sessions
UPDATE page_visits = arrayPushBack(page_visits, 'pricing')
WHERE user_id = 1;

-- After update:
-- user 1: ['home', 'about', 'pricing']
-- user 2: ['landing']
```

## Maintaining Bounded History Arrays

Combining `arrayPushBack` with `arraySlice` lets you maintain a rolling window - append a new item and truncate to keep only the last N entries:

```sql
CREATE TABLE rolling_scores
(
    player_id UInt32,
    last_10_scores Array(UInt32)
) ENGINE = MergeTree()
ORDER BY player_id;

INSERT INTO rolling_scores VALUES
    (1, [85, 90, 78, 92, 88, 76, 94, 81, 87, 83]),
    (2, [70, 75]);

-- Add a new score and keep only the last 10
ALTER TABLE rolling_scores
UPDATE last_10_scores = arraySlice(
    arrayPushBack(last_10_scores, 95),
    -10
)
WHERE player_id = 1;

-- player 1's oldest score (85) is dropped, 95 is added at the end
```

## Building Path Arrays for Hierarchy Traversal

When traversing a hierarchy or building breadcrumb paths, `arrayPushBack` builds the path as you descend each level:

```sql
-- Simulate building a file path array level by level
SELECT
    arrayPushBack(
        arrayPushBack(
            arrayPushBack(['/', 'usr'], 'local'),
            'bin'
        ),
        'clickhouse'
    ) AS path_parts;
-- Result: ['/', 'usr', 'local', 'bin', 'clickhouse']

-- Join into a string path
SELECT arrayStringConcat(
    arrayPushBack(
        arrayPushBack(['usr', 'local'], 'bin'),
        'clickhouse'
    ),
    '/'
) AS full_path;
-- Result: 'usr/local/bin/clickhouse'
```

## Prepending Timestamps or Sequence Numbers

`arrayPushFront` is useful for maintaining reverse-chronological order: prepend new items so the most recent is always at index 1:

```sql
-- Prepend newest event to front so array[1] is always most recent
ALTER TABLE user_sessions
UPDATE page_visits = arrayPushFront(page_visits, 'checkout')
WHERE user_id = 1;

-- user 1: ['checkout', 'home', 'about', 'pricing']
-- Accessing array[1] gives the latest page
SELECT
    user_id,
    page_visits[1] AS most_recent_page
FROM user_sessions;
```

## Adding Sentinel or Boundary Values

Both functions are handy for adding sentinel values at the edges of arrays before further processing. For example, adding a 0 at the start before computing differences:

```sql
-- Add a leading 0 so arrayDifference has a meaningful first delta
SELECT arrayDifference(
    arrayPushFront([10, 25, 40, 55], 0)
) AS deltas_from_zero;
-- Input to arrayDifference: [0, 10, 25, 40, 55]
-- Result: [0, 10, 15, 15, 15]
```

## Summary

`arrayPushBack` and `arrayPushFront` are the fundamental building blocks for extending arrays in ClickHouse. They append or prepend a single element and return a new array, which can then be chained, sliced, or used in mutations. Combined with `arraySlice`, they enable bounded rolling windows; combined with `ALTER UPDATE`, they let you maintain ever-growing event log arrays in MergeTree tables.
