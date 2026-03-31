# How to Use arrayMap() and arrayFilter() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array Function, arrayMap, arrayFilter, Lambda, Higher-Order Function

Description: Learn how arrayMap() transforms every element with a lambda and arrayFilter() keeps only elements matching a predicate, with practical SQL examples.

---

ClickHouse supports higher-order functions that accept lambda expressions as arguments. Two of the most commonly used are `arrayMap()` and `arrayFilter()`. `arrayMap()` applies a transformation to every element of an array and returns a new array of results. `arrayFilter()` returns a new array containing only the elements for which a predicate lambda returns a non-zero value. Together they let you process array columns with the same expressiveness you would expect from a functional programming language, all inside a SQL query.

## Lambda Syntax in ClickHouse

Before diving into examples, it is important to understand how ClickHouse lambda expressions are written. The syntax is:

```text
element -> expression
```

For functions that need to operate on multiple arrays in parallel, you can use a tuple of parameter names:

```text
(x, y) -> expression
```

## Transforming Elements with arrayMap()

`arrayMap()` passes each element through a lambda and collects the results. The input and output arrays always have the same length.

```sql
-- Double every number in the array
SELECT arrayMap(x -> x * 2, [1, 2, 3, 4, 5]) AS doubled;
-- Result: [2, 4, 6, 8, 10]

-- Convert log level codes to human-readable strings
SELECT arrayMap(
    code -> multiIf(code = 1, 'debug', code = 2, 'info', code = 3, 'warn', 'error'),
    [1, 2, 3, 4]
) AS levels;
```

## Applying arrayMap() to a Table Column

When the array is stored in a column, `arrayMap()` works identically - it simply operates on the per-row array value.

```sql
-- Normalize all tags to lowercase
SELECT
    user_id,
    arrayMap(t -> lower(t), tags) AS normalized_tags
FROM events;

-- Add a fixed prefix to every item in a list
SELECT
    order_id,
    arrayMap(sku -> concat('SKU-', sku), line_items) AS prefixed_skus
FROM orders;
```

## Filtering Elements with arrayFilter()

`arrayFilter()` evaluates the lambda for each element and retains only those where the result is non-zero (truthy).

```sql
-- Keep only positive numbers
SELECT arrayFilter(x -> x > 0, [-3, -1, 0, 2, 5, 8]) AS positives;
-- Result: [2, 5, 8]

-- Keep only error-level log codes (>= 3)
SELECT
    request_id,
    arrayFilter(code -> code >= 3, event_codes) AS error_codes
FROM request_log;
```

## Combining arrayMap() and arrayFilter()

A common pattern is to filter first and then transform, or transform first and then filter. Both orderings are valid depending on whether the transformation simplifies the predicate.

```sql
-- Filter to keep scores above 50, then scale them to percentages
SELECT arrayMap(
    s -> s / 100.0,
    arrayFilter(s -> s > 50, [20, 55, 70, 45, 90])
) AS high_score_pct;

-- Alternatively: map first, then filter on the transformed value
SELECT arrayFilter(
    p -> p > 0.5,
    arrayMap(s -> s / 100.0, [20, 55, 70, 45, 90])
) AS high_score_pct_v2;
```

## Using Multi-Array Lambdas with arrayMap()

When you pass multiple arrays to `arrayMap()`, the lambda receives one element from each array simultaneously. All arrays must be the same length.

```sql
-- Compute element-wise product of two arrays
SELECT arrayMap((a, b) -> a * b, [1, 2, 3], [10, 20, 30]) AS products;
-- Result: [10, 40, 90]

-- Build a formatted string from two parallel arrays
SELECT arrayMap(
    (name, score) -> concat(name, ': ', toString(score)),
    ['alice', 'bob', 'carol'],
    [95, 82, 78]
) AS leaderboard;
```

## Practical Example: Clamping Values

`arrayMap()` can implement value clamping across all elements without a loop.

```sql
-- Clamp latency values to a maximum of 1000ms
SELECT
    request_id,
    arrayMap(v -> least(v, 1000), latency_samples) AS clamped_latencies
FROM performance_log;
```

## Practical Example: Removing Empty Strings from Tag Arrays

A common data-quality step is to strip empty or whitespace-only strings from array columns.

```sql
SELECT
    user_id,
    arrayFilter(t -> notEmpty(trim(t)), raw_tags) AS clean_tags
FROM user_profiles;
```

## Practical Example: Extracting Domain from URL Arrays

`arrayMap()` can apply any scalar ClickHouse function, including string parsing functions.

```sql
-- Extract hostname from each URL stored in an array column
SELECT
    session_id,
    arrayMap(url -> domain(url), visited_urls) AS domains
FROM sessions;
```

## Counting Filtered Results with length()

Wrapping `arrayFilter()` with `length()` is an efficient way to count elements satisfying a condition without unnesting the array.

```sql
-- Count how many events per session had a duration over 5 seconds
SELECT
    session_id,
    length(arrayFilter(d -> d > 5000, event_durations_ms)) AS slow_event_count
FROM sessions;
```

## Summary

`arrayMap()` and `arrayFilter()` bring functional programming primitives into ClickHouse SQL. `arrayMap()` transforms every element of an array through a lambda, making it straightforward to normalize, reformat, or compute derived values across all elements at once. `arrayFilter()` selects only the elements that satisfy a predicate, enabling inline data cleaning and conditional extraction. Because both functions work on the array as a whole rather than unnesting rows, they are faster and simpler than alternatives based on `arrayJoin()` and re-aggregation.
