# How to Use arrayDistinct() and arrayUniq() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array Function, arrayDistinct, arrayUniq, Deduplication, SQL

Description: Learn how arrayDistinct() removes duplicate elements preserving first-occurrence order and arrayUniq() counts distinct tuples across arrays, with practical examples.

---

Deduplication is a common step in array processing pipelines. ClickHouse provides two functions with complementary roles: `arrayDistinct(arr)` returns a new array with duplicates removed, preserving the order of first occurrences; `arrayUniq(arr1, arr2, ...)` counts the number of distinct element tuples across one or more parallel arrays. Knowing which function to reach for depends on whether you need the deduplicated array itself or just the count of unique values or combinations.

## arrayDistinct() - Remove Duplicates from an Array

`arrayDistinct(arr)` scans the array and keeps only the first occurrence of each value, returning a new array in the same relative order.

```sql
-- Remove duplicate integers
SELECT arrayDistinct([1, 2, 3, 2, 1, 4, 3, 5]) AS unique_values;
-- Result: [1, 2, 3, 4, 5]

-- Remove duplicate strings, preserving first-seen order
SELECT arrayDistinct(['error', 'warn', 'error', 'info', 'warn']) AS unique_levels;
-- Result: ['error', 'warn', 'info']
```

## Deduplicating Tag Arrays from a Table Column

When tag arrays are assembled from multiple sources they often contain duplicates. `arrayDistinct()` cleans them up.

```sql
-- Remove duplicate tags from each event's merged tag list
SELECT
    event_id,
    arrayDistinct(all_tags) AS unique_tags
FROM events;

-- Merge two tag sources and deduplicate in one step
SELECT
    event_id,
    arrayDistinct(arrayConcat(source_a_tags, source_b_tags)) AS merged_unique_tags
FROM events;
```

## Counting Unique Elements in an Array

Wrapping `arrayDistinct()` with `length()` gives the number of distinct values, which is the array equivalent of `COUNT(DISTINCT ...)`.

```sql
-- Count distinct tags per event
SELECT
    event_id,
    length(arrayDistinct(tags)) AS unique_tag_count
FROM events;

-- Count distinct status codes observed per session
SELECT
    session_id,
    length(arrayDistinct(status_codes)) AS distinct_status_count
FROM session_log;
```

## arrayUniq() - Count Distinct Values Across One or More Arrays

`arrayUniq(arr)` returns the count of distinct values in a single array. It is equivalent to `length(arrayDistinct(arr))` for a single array but more concise.

```sql
-- Count distinct values in one array
SELECT arrayUniq([1, 2, 3, 2, 1]) AS distinct_count;
-- Result: 3
```

## arrayUniq() with Multiple Arrays - Distinct Tuple Counting

When passed two or more arrays of equal length, `arrayUniq(arr1, arr2)` counts the number of distinct element-tuples formed by zipping the arrays together. This is useful for counting unique (key, value) pairs or unique combinations of attributes.

```sql
-- Count distinct (service, status_code) pairs in parallel arrays
SELECT
    trace_id,
    arrayUniq(span_services, span_status_codes) AS distinct_service_status_pairs
FROM distributed_traces;
```

## Comparing arrayDistinct() vs. arrayUniq()

The key difference is that `arrayDistinct()` returns the deduplicated array while `arrayUniq()` returns only the count. Use `arrayDistinct()` when you need the values; use `arrayUniq()` when you only need the number.

```sql
-- Get the actual unique tags AND the count
SELECT
    event_id,
    arrayDistinct(tags) AS unique_tags,
    arrayUniq(tags) AS unique_tag_count
FROM events;
```

## Deduplicating Before Aggregation

When collecting values with `groupArray()` into an array for further processing, wrapping the result with `arrayDistinct()` avoids counting items multiple times.

```sql
-- Collect distinct user-visited page IDs across all sessions in a cohort
SELECT
    cohort_id,
    arrayDistinct(groupArray(page_id)) AS distinct_pages_visited
FROM (
    SELECT cohort_id, arrayJoin(visited_pages) AS page_id
    FROM user_sessions
)
GROUP BY cohort_id;
```

## Detecting Arrays With All Unique Elements

Combining `length()` with `arrayDistinct()` lets you check whether an array already contains only unique values.

```sql
-- Flag arrays that have no duplicates
SELECT
    order_id,
    length(line_item_skus) = length(arrayDistinct(line_item_skus)) AS no_duplicate_skus
FROM orders;

-- Find orders with duplicate SKUs (possible data issue)
SELECT order_id
FROM orders
WHERE length(line_item_skus) != length(arrayDistinct(line_item_skus));
```

## Deduplicating Sorted Arrays Efficiently with arrayCompact()

For arrays that are already sorted, `arrayCompact()` removes consecutive duplicates in a single pass and is more efficient than `arrayDistinct()` in that specific scenario.

```sql
-- For pre-sorted arrays, arrayCompact is more efficient than arrayDistinct
SELECT arrayCompact(arraySort([1, 1, 2, 3, 3, 3, 4])) AS compact;
-- Result: [1, 2, 3, 4]

-- arrayDistinct would work too but does more work for sorted input
SELECT arrayDistinct(arraySort([1, 1, 2, 3, 3, 3, 4])) AS distinct;
-- Result: [1, 2, 3, 4]
```

## Practical Example: Unique Services in a Distributed Trace

In distributed tracing, a single trace may visit the same service multiple times. `arrayDistinct()` on the span services array gives the unique set of services involved.

```sql
SELECT
    trace_id,
    span_services,
    arrayDistinct(span_services) AS unique_services,
    length(arrayDistinct(span_services)) AS service_count
FROM distributed_traces
ORDER BY service_count DESC
LIMIT 10;
```

## Practical Example: Top Distinct Tags Across the Dataset

Combining `arrayJoin()`, `arrayDistinct()`, and aggregation lets you find which distinct tags appear most frequently across all rows.

```sql
-- Count how many events each unique tag appears in (one count per event even if tag repeats within it)
SELECT
    tag,
    count() AS event_count
FROM (
    SELECT
        event_id,
        arrayJoin(arrayDistinct(tags)) AS tag
    FROM events
)
GROUP BY tag
ORDER BY event_count DESC
LIMIT 20;
```

## Summary

`arrayDistinct()` removes duplicate elements from an array while preserving the first-occurrence order, returning the deduplicated array itself. `arrayUniq()` returns only the count of distinct values, and when called with multiple arrays it counts distinct element-tuples across them. Use `arrayDistinct()` when you need the cleaned array for further processing or display; use `arrayUniq()` when a cardinality count is all you need. For pre-sorted arrays, `arrayCompact()` is a faster alternative to `arrayDistinct()` for removing consecutive duplicates.
