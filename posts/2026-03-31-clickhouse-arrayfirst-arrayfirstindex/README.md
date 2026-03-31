# How to Use arrayFirst() and arrayFirstIndex() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array Function, arrayFirst, arrayFirstIndex, Lambda, Higher-Order Function

Description: Learn how arrayFirst() retrieves the first element satisfying a lambda and arrayFirstIndex() returns its 1-based position, with practical SQL examples.

---

When processing array columns in ClickHouse you frequently need to locate the first element that satisfies some condition - the first error in a log sequence, the first score above a threshold, or the first tag matching a category. `arrayFirst()` returns that element directly, while `arrayFirstIndex()` returns its 1-based position within the array. Both accept a lambda predicate and short-circuit as soon as a match is found, making them efficient even on long arrays.

## arrayFirst() - Return the First Matching Element

`arrayFirst(func, arr)` evaluates the lambda for each element from left to right and returns the first element for which the lambda is non-zero. If no element matches, it returns the zero value for the element type (0 for numeric types, an empty string for strings).

```sql
-- Return the first positive number
SELECT arrayFirst(x -> x > 0, [-5, -2, 3, 7, -1]) AS first_positive;
-- Result: 3

-- Return the first string that starts with 'err'
SELECT arrayFirst(s -> startsWith(s, 'err'), ['warn', 'error', 'info', 'error']) AS first_error;
-- Result: 'error'
```

## arrayFirstIndex() - Return the Position of the First Match

`arrayFirstIndex(func, arr)` returns the 1-based index of the first matching element. If no element matches, it returns `0`.

```sql
-- Find the position of the first positive number
SELECT arrayFirstIndex(x -> x > 0, [-5, -2, 3, 7, -1]) AS first_positive_idx;
-- Result: 3

-- Find the position of the first 'error' entry
SELECT arrayFirstIndex(s -> s = 'error', ['warn', 'error', 'info', 'error']) AS first_error_idx;
-- Result: 2
```

## Locating the First Error in a Request Sequence

A typical observability pattern stores a sequence of HTTP status codes for each session. `arrayFirst()` and `arrayFirstIndex()` let you identify where things went wrong.

```sql
-- Find the first error status code in each session's request sequence
SELECT
    session_id,
    arrayFirst(s -> s >= 400, status_codes) AS first_error_code,
    arrayFirstIndex(s -> s >= 400, status_codes) AS first_error_position
FROM session_log;
```

## Finding the First Event Above a Latency Threshold

```sql
-- Identify which request in a sequence first exceeded 1000ms
SELECT
    trace_id,
    arrayFirstIndex(d -> d > 1000, span_durations_ms) AS slow_span_index,
    arrayFirst(d -> d > 1000, span_durations_ms) AS slow_span_ms
FROM distributed_traces;
```

## Checking for No Match

When `arrayFirstIndex()` returns `0`, it means no element matched. This makes it easy to distinguish "found" from "not found."

```sql
-- Flag sessions where no error occurred (no 5xx in the sequence)
SELECT
    session_id,
    arrayFirstIndex(s -> s >= 500, status_codes) = 0 AS fully_successful
FROM session_log;
```

## Extracting the First Matching Tag

In tag-rich datasets, you often need the first tag from a preferred category rather than the full list.

```sql
-- Extract the first severity tag from each incident's tag array
SELECT
    incident_id,
    arrayFirst(
        t -> t IN ('critical', 'high', 'medium', 'low'),
        tags
    ) AS severity_tag
FROM incidents;
```

## Using arrayFirstIndex() to Slice an Array

Because `arrayFirstIndex()` returns a position, you can use it together with array slicing functions to extract everything from the first match onward.

```sql
-- Get all status codes starting from the first error
SELECT
    session_id,
    arraySlice(
        status_codes,
        arrayFirstIndex(s -> s >= 400, status_codes)
    ) AS codes_from_first_error
FROM session_log
WHERE arrayFirstIndex(s -> s >= 400, status_codes) > 0;
```

## Multi-Array Lambda

Both functions support multi-array lambdas, receiving one element from each array simultaneously.

```sql
-- Find the first span where duration exceeds its budget
SELECT
    trace_id,
    arrayFirstIndex(
        (duration, budget) -> duration > budget,
        span_durations_ms,
        span_budgets_ms
    ) AS first_over_budget_index
FROM distributed_traces;
```

## Combining with arrayMap() to Get Context

When you need more context around the first match, `arrayFirstIndex()` combined with direct array element access gives you access to any parallel column.

```sql
-- Get the service name of the span that first exceeded budget
SELECT
    trace_id,
    arrayFirstIndex((d, b) -> d > b, span_durations_ms, span_budgets_ms) AS idx,
    span_service_names[
        arrayFirstIndex((d, b) -> d > b, span_durations_ms, span_budgets_ms)
    ] AS offending_service
FROM distributed_traces
WHERE arrayFirstIndex((d, b) -> d > b, span_durations_ms, span_budgets_ms) > 0;
```

## Summary

`arrayFirst()` and `arrayFirstIndex()` are targeted search functions that scan an array left to right and stop at the first element satisfying a lambda predicate. `arrayFirst()` returns the element value, while `arrayFirstIndex()` returns its 1-based position (or `0` when no match exists). Both are efficient due to short-circuit evaluation and are well suited for finding the first anomaly in an event sequence, extracting the leading matching tag, or slicing arrays from a known point of interest.
