# How to Use indexOf() for Array Search in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array Function, indexOf, Array Search, SQL

Description: Learn how indexOf() finds the 1-based position of an element in an array, returning 0 when absent, with examples for lookup logic and element ordering.

---

`indexOf(arr, elem)` searches an array from left to right and returns the 1-based index of the first occurrence of `elem`. If the element is not found, the function returns `0`. This makes `indexOf()` the array equivalent of a membership test combined with position lookup: the return value tells you both whether the element exists (non-zero means yes) and exactly where it is. It is simpler than `arrayFirstIndex()` for exact-value searches because you do not need a lambda.

## Basic Usage

```sql
-- Find the position of value 30 in the array
SELECT indexOf([10, 20, 30, 40, 50], 30) AS position;
-- Result: 3

-- Element not present - returns 0
SELECT indexOf([10, 20, 30, 40, 50], 99) AS position;
-- Result: 0

-- First occurrence is returned when there are duplicates
SELECT indexOf(['a', 'b', 'a', 'c'], 'a') AS first_a_pos;
-- Result: 1
```

## Checking Element Presence with indexOf()

Because `indexOf()` returns `0` when an element is absent, you can use it as a presence test without `has()` by comparing the result to `0`.

```sql
-- Find users who have the 'premium' flag (indexOf > 0 means present)
SELECT user_id
FROM user_profiles
WHERE indexOf(feature_flags, 'premium') > 0;

-- Find users who do NOT have the 'premium' flag
SELECT user_id
FROM user_profiles
WHERE indexOf(feature_flags, 'premium') = 0;
```

## Locating an Element's Position in a Stored Array

In many analytics use cases the position of an element in an ordered sequence is as important as its presence.

```sql
-- Find out at which step each user first encountered an error
SELECT
    session_id,
    indexOf(event_sequence, 'error') AS error_step
FROM sessions
WHERE indexOf(event_sequence, 'error') > 0;
```

## Checking Element Order Between Two Values

`indexOf()` returns numeric positions, so you can compare two positions to verify ordering.

```sql
-- Confirm that 'login' always appears before 'checkout' in the session
SELECT
    session_id,
    indexOf(event_sequence, 'login') AS login_step,
    indexOf(event_sequence, 'checkout') AS checkout_step,
    indexOf(event_sequence, 'login') < indexOf(event_sequence, 'checkout') AS correct_order
FROM sessions
WHERE indexOf(event_sequence, 'login') > 0
  AND indexOf(event_sequence, 'checkout') > 0;
```

## Computing Distance Between Two Elements

Subtracting two positions gives the number of steps between two events in a sequence.

```sql
-- How many steps separate 'add_to_cart' from 'payment' in each session?
SELECT
    session_id,
    indexOf(event_sequence, 'payment') - indexOf(event_sequence, 'add_to_cart') AS steps_to_payment
FROM sessions
WHERE indexOf(event_sequence, 'add_to_cart') > 0
  AND indexOf(event_sequence, 'payment') > 0;
```

## Using indexOf() to Extract a Paired Value from a Parallel Array

A common pattern in ClickHouse is storing two parallel arrays: one for keys and one for values. `indexOf()` on the keys array gives the index needed to retrieve the matching value.

```sql
-- Table: metrics (host String, metric_names Array(String), metric_values Array(Float64))
-- Retrieve the value for the 'cpu_idle' metric from each host's parallel arrays
SELECT
    host,
    metric_values[indexOf(metric_names, 'cpu_idle')] AS cpu_idle_pct
FROM metrics
WHERE indexOf(metric_names, 'cpu_idle') > 0;
```

## Building a Rank Column from a Priority List

When you have a canonical ordering of values (e.g., severity levels), `indexOf()` against that ordering array gives a numeric rank for each row's value.

```sql
-- Assign a numeric priority rank to each incident based on severity
WITH ['critical', 'high', 'medium', 'low', 'info'] AS severity_order
SELECT
    incident_id,
    severity,
    indexOf(severity_order, severity) AS priority_rank
FROM incidents
ORDER BY priority_rank;
```

## Filtering Based on Relative Position

You can filter rows based on when a specific event appears in a sequence - for example, only sessions where an error occurred in the first three steps.

```sql
-- Sessions where the first error was in the first three events
SELECT session_id
FROM sessions
WHERE indexOf(event_sequence, 'error') BETWEEN 1 AND 3;
```

## Combining indexOf() with arraySlice()

Once you have the position of a known element, `arraySlice()` can extract the portion of the array before or after it.

```sql
-- Extract all events that happened before the first error
SELECT
    session_id,
    arraySlice(event_sequence, 1, indexOf(event_sequence, 'error') - 1) AS pre_error_events
FROM sessions
WHERE indexOf(event_sequence, 'error') > 1;
```

## Summary

`indexOf()` is the simplest way to find the position of a specific value in a ClickHouse array. It returns a 1-based index on success and `0` when the element is absent, making it serve double duty as both a presence check and a position lookup. It is more concise than `arrayFirstIndex()` for exact-value searches. Key applications include element ordering verification, step-distance calculations, parallel-array key lookups, and dynamic array slicing from a known anchor point.
