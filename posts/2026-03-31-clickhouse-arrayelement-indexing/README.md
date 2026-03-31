# How to Use arrayElement() and Array Indexing in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array Function, Array Indexing, arrayElement, SQL

Description: Learn how arrayElement() and bracket notation access specific array positions, with examples covering first/last element extraction and fixed-schema arrays.

---

ClickHouse arrays are 1-based: the first element is at index `1`, not `0`. You access individual elements using either the `arrayElement(arr, n)` function or the equivalent bracket notation `arr[n]`. Negative indices count backward from the end of the array, so `arr[-1]` gives the last element. Out-of-bounds access returns the zero value for the element type rather than raising an error, which makes it safe to write defensive queries.

## Basic Indexing Syntax

Both the function form and the bracket shorthand produce identical results. The bracket form is generally more readable.

```sql
-- Access the second element using the function
SELECT arrayElement([10, 20, 30, 40], 2) AS second;
-- Result: 20

-- Equivalent bracket notation
SELECT [10, 20, 30, 40][2] AS second;
-- Result: 20

-- Negative index: last element
SELECT [10, 20, 30, 40][-1] AS last;
-- Result: 40

-- Negative index: second to last
SELECT [10, 20, 30, 40][-2] AS second_to_last;
-- Result: 30
```

## Out-of-Bounds Behavior

Accessing an index that does not exist returns the default zero value for the type. Always check the length before relying on a specific index if the array size is not guaranteed.

```sql
-- Index beyond array length returns 0 for numeric types
SELECT [1, 2, 3][10] AS out_of_bounds;
-- Result: 0

-- Out of bounds for String returns an empty string
SELECT ['a', 'b', 'c'][10] AS out_of_bounds_str;
-- Result: ''
```

## Extracting First and Last Elements from a Column

A common need is extracting the head or tail of a stored array. Negative indexing makes the "last element" pattern clean and readable.

```sql
-- Get the first and last status code from each session's sequence
SELECT
    session_id,
    status_codes[1] AS first_status,
    status_codes[-1] AS last_status
FROM session_log;
```

## Working with Fixed-Schema Arrays

When an array always has a known, fixed number of elements (for example, a row encoded as an array of metric values), indexing by position is an efficient way to unpack it.

```sql
-- Table stores [cpu_pct, mem_pct, disk_pct] as a fixed-width metrics array
SELECT
    host,
    metrics[1] AS cpu_pct,
    metrics[2] AS mem_pct,
    metrics[3] AS disk_pct
FROM host_snapshots;
```

## Using a Dynamic Index from Another Column

The index argument to `arrayElement()` does not have to be a literal - it can be any integer expression, including another column.

```sql
-- Each row stores which slot to read from in the values array
SELECT
    row_id,
    values[selected_index] AS selected_value
FROM dynamic_selections;
```

## Guarding Against Empty Arrays

When an array column might be empty, use `if(notEmpty(arr), arr[1], NULL)` or `length(arr) > 0` to avoid silently returning a zero default.

```sql
-- Safely extract the first element, returning NULL for empty arrays
SELECT
    event_id,
    if(notEmpty(tags), tags[1], NULL) AS primary_tag
FROM events;
```

## Accessing the Middle Element

Combining `length()` and integer division lets you compute the midpoint index dynamically.

```sql
-- Access the median-position element of each sorted sample array
SELECT
    sensor_id,
    readings[toUInt32(length(readings) / 2) + 1] AS middle_reading
FROM sensor_data;
```

## Extracting a Sub-array with arraySlice()

When you need more than one element from a known position, `arraySlice(arr, offset, length)` is more expressive than multiple index accesses.

```sql
-- Extract elements 2 through 4 from each row's array
SELECT
    trace_id,
    arraySlice(span_ids, 2, 3) AS middle_spans
FROM traces;

-- Extract the last 3 elements using a negative offset
SELECT
    trace_id,
    arraySlice(span_ids, -3) AS last_three_spans
FROM traces;
```

## Comparing First Elements Across Rows

Index access works in `ORDER BY`, `WHERE`, and join conditions just like any scalar expression.

```sql
-- Sort sessions by the latency of their first event
SELECT session_id, event_latencies_ms[1] AS first_latency
FROM sessions
ORDER BY first_latency DESC
LIMIT 10;

-- Filter rows where the first tag is 'critical'
SELECT event_id
FROM events
WHERE tags[1] = 'critical';
```

## Using arrayElement() in Aggregations

Because the indexed value is a scalar, you can aggregate it directly.

```sql
-- Average the first-event latency across all sessions
SELECT avg(event_latencies_ms[1]) AS avg_first_latency
FROM sessions;

-- Count sessions where the last status code indicates an error
SELECT count() AS error_ending_sessions
FROM sessions
WHERE status_codes[-1] >= 400;
```

## Summary

`arrayElement()` and its bracket equivalent `arr[n]` provide direct positional access to ClickHouse array elements using 1-based indices. Negative indices count from the end of the array, making first and last element extraction concise. Out-of-bounds access returns a safe zero default rather than an error, but you should guard against empty arrays explicitly when the default would be misleading. For multi-element extraction, prefer `arraySlice()` over multiple index calls.
