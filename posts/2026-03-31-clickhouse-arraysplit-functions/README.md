# How to Use arraySplit() and arrayReverseSplit() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array Function, arraySplit, arrayReverseSplit, Session Analysis

Description: Learn how arraySplit() and arrayReverseSplit() divide ClickHouse arrays into sub-arrays at positions defined by a lambda, enabling session detection and array chunking.

---

Splitting an array into sub-arrays at dynamic positions is essential for session analysis, chunking event streams, and segmenting time-series data. `arraySplit` divides an array wherever a lambda condition returns 1, while `arrayReverseSplit` does the same from right to left. The result is an array of arrays, ready for further per-segment analysis.

## Function Signatures

```text
arraySplit(func, arr1 [, arr2, ...])        -> Array(Array(T))
arrayReverseSplit(func, arr1 [, arr2, ...]) -> Array(Array(T))
```

The lambda `func` receives the current element and returns 1 to start a new sub-array **before** the current element, or 0 to include the current element in the current sub-array. The element where the lambda returns 1 becomes the **first element of the new sub-array** - it is not discarded.

## Basic Usage

```sql
-- Split at every even number (start new chunk before each even number)
SELECT arraySplit(x -> (x % 2 = 0), [1, 3, 2, 4, 5, 7, 6]) AS chunks;
-- x=1: 0 (no split), x=3: 0 (no split), x=2: 1 (split!), x=4: 1 (split!), ...
-- Result: [[1,3],[2],[4],[5,7],[6]]

-- Split at every position where value decreases
SELECT arraySplit(
    (cur, prev) -> (cur < prev),
    [1, 3, 5, 2, 4, 6, 1, 3]
) AS runs;
-- Result: [[1,3,5],[2,4,6],[1,3]]  (each ascending run is one sub-array)
```

Note: the two-argument lambda receives `(current, previous)` element for context-aware splitting.

## Sessionizing an Event Stream

The most valuable application of `arraySplit` is detecting session boundaries in a time-ordered event array. A new session starts when the gap between consecutive timestamps exceeds a threshold:

```sql
CREATE TABLE user_clickstreams
(
    user_id UInt32,
    -- Unix timestamps of events in seconds
    event_times Array(UInt32),
    event_pages Array(String)
) ENGINE = Memory;

INSERT INTO user_clickstreams VALUES
    (1,
     [1700000000, 1700000060, 1700000120,  -- session 1 (2 min apart)
      1700003700,                           -- gap > 30 min -> session 2
      1700003760, 1700003840],             -- session 2 continues
     ['home', 'about', 'pricing', 'home', 'checkout', 'confirm']),
    (2,
     [1700010000, 1700010300, 1700014000], -- two sessions
     ['blog', 'post', 'home']);

-- Split timestamps into sessions (new session if gap > 1800 seconds = 30 min)
SELECT
    user_id,
    arraySplit(
        (cur, prev) -> (cur - prev > 1800),
        event_times
    ) AS sessions
FROM user_clickstreams;
-- user 1: [[1700000000,1700000060,1700000120],[1700003700,1700003760,1700003840]]
-- user 2: [[1700010000,1700010300],[1700014000]]

-- Count sessions per user
SELECT
    user_id,
    length(arraySplit(
        (cur, prev) -> (cur - prev > 1800),
        event_times
    )) AS num_sessions
FROM user_clickstreams;
```

## Splitting Pages Into Sessions Simultaneously

Use parallel arrays in the lambda to split the page array at the same positions as the time array:

```sql
-- Split both timestamps and pages using the time-based session boundary
SELECT
    user_id,
    arraySplit(
        (t_cur, t_prev) -> (t_cur - t_prev > 1800),
        event_times,    -- drive the split condition
        event_times     -- also split the times (redundant here, shown for pattern)
    ) AS session_times,
    arraySplit(
        (t_cur, t_prev) -> (t_cur - t_prev > 1800),
        event_times,
        event_pages
    ) AS session_pages
FROM user_clickstreams;
```

Note: in `arraySplit(func, arr1, arr2)`, the lambda receives elements from all arrays, but the output is the split of the last array argument.

## Chunking Arrays into Fixed-Size Blocks

Split an array into chunks of exactly N elements using a counter:

```sql
-- Split [1..10] into chunks of 3 using arrayEnumerate
SELECT arraySplit(
    (val, idx) -> (idx % 3 = 1 AND idx != 1),
    range(1, 11),        -- values
    range(1, 11)         -- indices (same here for simplicity)
) AS chunked;
-- Split starts at positions 4, 7, 10 (every 3rd starting from 4)
-- Result: [[1,2,3],[4,5,6],[7,8,9],[10]]
```

A cleaner approach for fixed-size chunking uses `arrayEnumerate`:

```sql
WITH [10, 20, 30, 40, 50, 60, 70, 80, 90] AS arr
SELECT arraySplit(
    (v, i) -> (i % 3 = 1 AND i > 1),
    arr,
    arrayEnumerate(arr)
) AS chunks_of_3;
-- Result: [[10,20,30],[40,50,60],[70,80,90]]
```

## arrayReverseSplit - Splitting from Right to Left

`arrayReverseSplit` applies the same logic but scans from the end of the array. A split starts a new sub-array to the **right** of the split position. This is useful when you want to identify the last N items before a condition:

```sql
-- Split from right: new chunk starts right of each increasing value
SELECT arrayReverseSplit(
    (cur, prev) -> (cur > prev),
    [1, 3, 5, 2, 4, 6]
) AS reverse_chunks;
-- Result: [[1,3,5],[2,4],[6]]
-- Right-to-left: 6 > 4 -> split, 4 > 2 -> split, no more splits

-- Count elements in the last session (most recent)
WITH arraySplit(
    (cur, prev) -> (cur - prev > 1800),
    event_times
) AS sessions
SELECT
    user_id,
    length(sessions[length(sessions)]) AS last_session_events
FROM user_clickstreams;
```

## Analyzing Per-Session Statistics

After splitting into sessions, apply `arrayReduce` or `arrayMap` to each session sub-array:

```sql
WITH arraySplit(
    (cur, prev) -> (cur - prev > 1800),
    event_times
) AS sessions
SELECT
    user_id,
    length(sessions) AS num_sessions,
    arrayMap(s -> length(s), sessions) AS events_per_session,
    arrayMap(
        s -> s[length(s)] - s[1],
        sessions
    ) AS session_durations_seconds
FROM user_clickstreams;
-- user 1: 2 sessions, [3,3] events each, [120, 140] second durations
-- user 2: 2 sessions, [2,1] events each, [300, 0] second durations
```

## Summary

`arraySplit` and `arrayReverseSplit` divide arrays into sub-arrays at positions where a lambda returns 1, returning `Array(Array(T))`. The split element begins the new sub-array rather than being discarded. These functions are the primary tool for sessionizing event streams by time gaps, chunking arrays into fixed-size blocks, and segmenting ordered sequences by any condition. After splitting, per-session metrics can be computed by mapping `arrayReduce`, `length`, or other functions over the resulting array of sub-arrays.
