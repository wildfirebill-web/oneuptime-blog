# How to Use array() and range() Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array Function, Range Function, SQL, Test Data

Description: Learn how array() creates arrays from literal values and range() generates integer sequences, with practical examples for test data and lookups.

---

ClickHouse provides two foundational functions for working with arrays from scratch: `array()` and `range()`. The `array()` function constructs an array directly from a list of values, while `range()` generates a sequence of integers. Together they cover the common need to create arrays inline, generate test datasets, and build lookup structures without having to store the data in a table first.

## Creating Arrays with array()

The `array()` function accepts any number of arguments of the same (or compatible) type and returns them packaged as a ClickHouse `Array(T)`. The shorthand bracket syntax `[v1, v2, ...]` is equivalent and often more readable.

```sql
-- Construct a simple array of integers
SELECT array(1, 2, 3, 4, 5) AS numbers;

-- The bracket shorthand is identical
SELECT [1, 2, 3, 4, 5] AS numbers;

-- Arrays of strings work the same way
SELECT array('info', 'warn', 'error') AS log_levels;

-- Mixed-compatible types are automatically promoted
SELECT array(1, 2.5, 3) AS mixed_nums;
```

## Generating Sequences with range()

`range(n)` produces the array `[0, 1, 2, ..., n-1]`. The three-argument form `range(start, end, step)` gives full control over the sequence. The result type is `Array(UInt)`, which fits naturally into arithmetic and looping patterns.

```sql
-- Generate [0, 1, 2, 3, 4]
SELECT range(5) AS seq;

-- Generate [1, 2, 3, 4, 5] using start/end
SELECT range(1, 6) AS seq;

-- Every other number: [0, 2, 4, 6, 8]
SELECT range(0, 10, 2) AS evens;

-- Countdown is not supported directly; reverse after generating
SELECT arrayReverse(range(1, 6)) AS countdown;
```

## Generating Test Data Rows

`range()` is particularly useful for generating synthetic rows on the fly by combining it with `arrayJoin()`. This avoids the need for a numbers table.

```sql
-- Generate 10 synthetic rows with an ID column
SELECT arrayJoin(range(1, 11)) AS id;

-- Generate timestamps spaced one hour apart starting from now
SELECT
    now() + toIntervalHour(arrayJoin(range(0, 24))) AS ts,
    'synthetic' AS source;
```

## Building Lookup Arrays

When you need to map an integer index to a label, constructing the label array inline with `array()` is cleaner than a CASE expression for long lists.

```sql
-- Map HTTP status codes to category labels using array indexing
WITH array('1xx', '2xx', '3xx', '4xx', '5xx') AS categories
SELECT
    status_code,
    categories[intDiv(status_code, 100)] AS category
FROM http_logs
WHERE status_code BETWEEN 100 AND 599;
```

## Using range() to Produce Fixed-Width Buckets

`range()` combined with `arrayMap()` can generate bucket boundary arrays used for histogram-style aggregations without writing repetitive CASE statements.

```sql
-- Build bucket boundaries [0, 100, 200, 300, 400, 500]
SELECT arrayMap(x -> x * 100, range(0, 6)) AS buckets;

-- Use those boundaries to assign each response_time to a bucket
WITH arrayMap(x -> x * 100, range(0, 6)) AS buckets
SELECT
    response_time,
    buckets[toUInt32(intDiv(response_time, 100)) + 1] AS bucket_start
FROM request_logs
WHERE response_time < 500;
```

## Combining array() and range() for Matrix-Style Data

When you need to generate positional arrays alongside value arrays, pairing `range()` with `array()` lets you zip an index onto an existing array.

```sql
-- Attach a 1-based index to each tag
WITH ['clickhouse', 'sql', 'arrays'] AS tags
SELECT
    arrayZip(range(1, length(tags) + 1), tags) AS indexed_tags;
```

## Practical Example: Generating a Weekly Schedule

Suppose you want to produce a row for each day of the current week. `range(7)` makes this straightforward.

```sql
SELECT
    today() + toIntervalDay(arrayJoin(range(7))) AS day,
    formatDateTime(today() + toIntervalDay(arrayJoin(range(7))), '%A') AS weekday;
```

## Summary

`array()` and `range()` are building blocks for array-oriented queries in ClickHouse. `array()` lets you construct arrays from literal values inline, avoiding the need for temporary tables or complex subqueries. `range()` generates integer sequences that serve as loop indices, bucket boundaries, and synthetic row generators. Mastering these two functions opens the door to the full suite of ClickHouse higher-order array functions.
