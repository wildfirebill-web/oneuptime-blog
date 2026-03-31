# How to Use arrayJoin() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array Function, arrayJoin, UNNEST, SQL

Description: Learn how arrayJoin() unnests array elements into individual rows, enabling powerful transformations that no ordinary function can achieve.

---

`arrayJoin()` is one of the most unique functions in ClickHouse. Unlike scalar functions that transform a value within a row, `arrayJoin()` changes the shape of the result set itself: it takes an array and emits one row for each element. This behavior is similar to the `UNNEST` keyword in other databases, but in ClickHouse it is expressed as a function call inside a `SELECT` list, making it composable with other expressions.

Understanding `arrayJoin()` is essential for working with event logs, time-series tag lists, and any schema where multiple values are packed into a single array column.

## Basic Usage: Unnesting an Array Literal

The simplest way to see `arrayJoin()` in action is to call it on a literal array. Each element becomes its own output row.

```sql
-- Produces three rows: 'info', 'warn', 'error'
SELECT arrayJoin(['info', 'warn', 'error']) AS log_level;
```

## Unnesting a Table Column

When a table stores an array column, `arrayJoin()` expands each row into as many rows as there are elements, while all other columns in that row are repeated for each element.

```sql
-- Assume a table: events (user_id UInt64, tags Array(String))
SELECT
    user_id,
    arrayJoin(tags) AS tag
FROM events;
```

This is the primary use case: normalizing a denormalized array column for aggregation or filtering.

## Filtering After Unnesting

Because `arrayJoin()` produces regular rows, you can filter on the unnested value with a standard `WHERE` clause.

```sql
-- Find all users who have the 'premium' tag
SELECT DISTINCT user_id
FROM events
WHERE arrayJoin(tags) = 'premium';
```

## Counting Elements Across All Arrays

Once rows are unnested, normal aggregate functions work without modification.

```sql
-- Count how many times each tag appears across all users
SELECT
    arrayJoin(tags) AS tag,
    count() AS occurrences
FROM events
GROUP BY tag
ORDER BY occurrences DESC;
```

## Unnesting with Index Using arrayEnumerate

If you need both the element and its position, combine `arrayJoin()` with `arrayEnumerate()`. The trick is to use `ARRAY JOIN` syntax (the clause form) or replicate the array in two places with a tuple approach. The cleanest method uses `arrayZip`.

```sql
-- Pair each tag with its 1-based position in the array
SELECT
    user_id,
    zipped.1 AS position,
    zipped.2 AS tag
FROM (
    SELECT
        user_id,
        arrayJoin(arrayZip(range(1, length(tags) + 1), tags)) AS zipped
    FROM events
);
```

## Cross-Joining Rows with a Fixed Array

`arrayJoin()` on a literal array effectively cross-joins every row in the table with the provided values - useful for generating multiple variants of each row.

```sql
-- Duplicate each log entry for each of three environments
SELECT
    event_id,
    event_name,
    arrayJoin(['production', 'staging', 'development']) AS environment
FROM event_log;
```

## Flattening Nested Arrays

When you have an `Array(Array(T))` column, a single `arrayJoin()` unwraps one level. Nesting two `arrayJoin()` calls (in a subquery) flattens two levels.

```sql
-- Table: batches (batch_id UInt32, items Array(Array(String)))
-- First unnest: get Array(String) rows
SELECT
    batch_id,
    arrayJoin(items) AS item_group
FROM batches;

-- Second unnest: fully flatten to individual strings
SELECT
    batch_id,
    arrayJoin(item) AS single_item
FROM (
    SELECT batch_id, arrayJoin(items) AS item
    FROM batches
);
```

## Unnesting Event Sequences for Gap Analysis

A practical observability pattern is to store a sequence of event timestamps in an array and then unnest to compute inter-event durations.

```sql
-- Table: sessions (session_id UInt64, event_times Array(DateTime))
SELECT
    session_id,
    arrayJoin(
        arrayMap(
            (t1, t2) -> t2 - t1,
            arrayPopBack(event_times),
            arrayPopFront(event_times)
        )
    ) AS gap_seconds
FROM sessions;
```

## Important Behavior: Multiple arrayJoin() Calls in One SELECT

Using `arrayJoin()` more than once in the same `SELECT` list produces a Cartesian product of the two arrays. This is rarely what you want; use the `ARRAY JOIN` clause or subqueries when joining multiple arrays from the same row.

```sql
-- WARNING: this produces a cross product
SELECT
    arrayJoin([1, 2]) AS a,
    arrayJoin([10, 20]) AS b;
-- Returns (1,10), (1,20), (2,10), (2,20)
```

## Summary

`arrayJoin()` is ClickHouse's primary tool for unnesting arrays into rows. It fundamentally reshapes query output rather than computing a scalar, which makes it unlike any other function in the language. Use it to normalize array columns for aggregation, to cross-join rows with a fixed set of values, and to flatten nested structures. When you need to unnest multiple arrays from the same row without a Cartesian product, reach for the `ARRAY JOIN` clause instead.
