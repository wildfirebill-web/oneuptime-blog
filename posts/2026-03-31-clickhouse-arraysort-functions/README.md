# How to Use arraySort() and arrayReverseSort() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array Function, arraySort, arrayReverseSort, Lambda, Higher-Order Function, SQL

Description: Learn how arraySort() sorts array elements ascending and arrayReverseSort() sorts descending, with lambda comparators and top-N extraction examples.

---

Sorting the elements within an array is a common need when you want to normalize array data, find extremes, or extract a ranked subset. ClickHouse provides `arraySort()` and `arrayReverseSort()` for in-place array sorting. Without a lambda, they sort the elements in their natural order. With a lambda, they sort by a computed key derived from each element, giving you full control over the sort criterion. Both functions return a new sorted array and leave the original unchanged.

## Basic Ascending Sort with arraySort()

`arraySort(arr)` returns the array elements in ascending natural order. For numbers that means smallest to largest; for strings it means lexicographic order.

```sql
-- Sort integers ascending
SELECT arraySort([3, 1, 4, 1, 5, 9, 2, 6]) AS sorted;
-- Result: [1, 1, 2, 3, 4, 5, 6, 9]

-- Sort strings lexicographically
SELECT arraySort(['banana', 'apple', 'cherry']) AS sorted;
-- Result: ['apple', 'banana', 'cherry']
```

## Descending Sort with arrayReverseSort()

`arrayReverseSort(arr)` returns elements in descending order. This is equivalent to `arrayReverse(arraySort(arr))` but slightly more expressive.

```sql
-- Sort integers descending
SELECT arrayReverseSort([3, 1, 4, 1, 5, 9, 2, 6]) AS sorted_desc;
-- Result: [9, 6, 5, 4, 3, 2, 1, 1]

-- Sort strings reverse-lexicographically
SELECT arrayReverseSort(['banana', 'apple', 'cherry']) AS sorted_desc;
-- Result: ['cherry', 'banana', 'apple']
```

## Sorting by a Lambda Comparator

The lambda form `arraySort(func, arr)` sorts by the key returned by the lambda rather than by the element value directly. This is useful when elements are complex (e.g., strings that encode numeric values) or when you want to sort by a derived property.

```sql
-- Sort strings by their length, shortest first
SELECT arraySort(s -> length(s), ['banana', 'fig', 'apple', 'kiwi']) AS by_length;
-- Result: ['fig', 'kiwi', 'apple', 'banana']

-- Sort version strings by their numeric suffix
SELECT arraySort(v -> toUInt32(extractAll(v, '[0-9]+')[1]), ['v3', 'v10', 'v2', 'v11']) AS by_version;
-- Result: ['v2', 'v3', 'v10', 'v11']
```

## Applying arraySort() to Table Columns

Sort operations apply per row when the array is a column value.

```sql
-- Normalize each session's event duration list to ascending order
SELECT
    session_id,
    arraySort(event_durations_ms) AS sorted_durations
FROM sessions;
```

## Extracting the Top-N Elements Without GROUP BY

Combining `arrayReverseSort()` with `arraySlice()` extracts the largest N elements from any stored array without touching `GROUP BY` or window functions.

```sql
-- Get the top 3 latencies from each trace's span durations
SELECT
    trace_id,
    arraySlice(arrayReverseSort(span_durations_ms), 1, 3) AS top3_latencies_ms
FROM distributed_traces;
```

## Extracting the Bottom-N Elements

The same pattern with `arraySort()` gives the N smallest values.

```sql
-- Get the 5 lowest scores for anomaly review
SELECT
    experiment_id,
    arraySlice(arraySort(scores), 1, 5) AS bottom5_scores
FROM experiment_results;
```

## Sorting a Parallel Array by Another Array's Order

A powerful pattern is sorting a values array by a separate key array. Use the multi-array lambda form to achieve this.

```sql
-- Sort span names by their corresponding durations (ascending)
SELECT
    trace_id,
    arraySort(
        (name, duration) -> duration,
        span_names,
        span_durations_ms
    ) AS spans_by_duration
FROM distributed_traces;
```

## Sorting Descending with a Lambda

`arrayReverseSort()` also accepts a lambda. Use it to sort descending by a computed key.

```sql
-- Sort tags by frequency (most common tag first) using a precomputed count array
SELECT
    arrayReverseSort(
        (tag, cnt) -> cnt,
        ['clickhouse', 'sql', 'arrays', 'performance'],
        [120, 340, 89, 210]
    ) AS tags_by_popularity;
-- Result: ['sql', 'performance', 'clickhouse', 'arrays']
```

## Deduplicating After Sorting

`arraySort()` followed by `arrayCompact()` removes consecutive duplicates, which after sorting means all duplicates are removed.

```sql
-- Sort and deduplicate an array of status codes
SELECT
    session_id,
    arrayCompact(arraySort(status_codes)) AS unique_sorted_statuses
FROM session_log;
```

## Ranking Elements by Position After Sort

Knowing the sorted order but also needing to map back to original positions requires combining `arraySort()` with `arrayEnumerate()` in a creative way. A practical alternative is to `arrayZip` the elements with their indices before sorting.

```sql
-- Get (original_index, value) pairs sorted by value descending
SELECT
    trace_id,
    arraySort(
        (pair) -> -pair.2,
        arrayZip(range(1, length(span_durations_ms) + 1), span_durations_ms)
    ) AS ranked_spans
FROM distributed_traces;
```

## Summary

`arraySort()` and `arrayReverseSort()` sort the elements of an array in ascending and descending order respectively. The optional lambda argument unlocks sorting by a derived key, enabling sorts by string length, numeric suffix, or any computable property. Common applications include normalizing stored arrays, extracting top-N or bottom-N values with `arraySlice()`, and sorting one array by the ordering defined by a parallel companion array. Both functions return a new array, leaving the source data unchanged.
