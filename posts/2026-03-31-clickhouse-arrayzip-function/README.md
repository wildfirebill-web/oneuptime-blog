# How to Use arrayZip() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array Function, arrayZip, Tuple

Description: Learn how arrayZip() combines multiple parallel arrays element-wise into an array of tuples in ClickHouse for pairing keys with values and transposing row arrays.

---

When you store parallel arrays - one for keys and one for values, or several metrics recorded at the same timestamps - you often want to process them as paired tuples. `arrayZip` takes two or more arrays and combines them element-wise into a single array of tuples. This is the ClickHouse equivalent of Python's `zip()` built-in.

## Function Signature

```text
arrayZip(arr1, arr2 [, arr3, ...]) -> Array(Tuple(T1, T2, ...))
```

All input arrays must have the same length. The result is an array of tuples where the i-th tuple contains the i-th element from each input array.

## Basic Usage

```sql
-- Zip two arrays into key-value pairs
SELECT arrayZip(['a', 'b', 'c'], [1, 2, 3]) AS zipped;
-- Result: [('a', 1), ('b', 2), ('c', 3)]

-- Zip three arrays
SELECT arrayZip([1, 2, 3], ['x', 'y', 'z'], [true, false, true]) AS triple_zip;
-- Result: [(1,'x',1), (2,'y',0), (3,'z',1)]

-- Accessing tuple elements after zip
SELECT
    t.1 AS key,
    t.2 AS value
FROM (
    SELECT arrayJoin(arrayZip(['cpu', 'mem', 'disk'], [80.5, 62.1, 45.3])) AS t
);
-- cpu  80.5
-- mem  62.1
-- disk 45.3
```

## Pairing Metric Names with Values

A common pattern in observability is storing metric names and metric values as separate parallel arrays. `arrayZip` unifies them into structured tuples for cleaner processing:

```sql
CREATE TABLE metric_snapshots
(
    host String,
    metric_names Array(String),
    metric_values Array(Float64)
) ENGINE = Memory;

INSERT INTO metric_snapshots VALUES
    ('server-01', ['cpu_pct', 'mem_pct', 'disk_pct'], [78.2, 55.1, 42.9]),
    ('server-02', ['cpu_pct', 'mem_pct', 'disk_pct'], [12.4, 88.7, 91.3]);

-- Zip and unnest to get one row per metric per host
SELECT
    host,
    metric.1 AS metric_name,
    metric.2 AS metric_value
FROM metric_snapshots
ARRAY JOIN arrayZip(metric_names, metric_values) AS metric;
-- server-01  cpu_pct   78.2
-- server-01  mem_pct   55.1
-- server-01  disk_pct  42.9
-- server-02  cpu_pct   12.4
-- server-02  mem_pct   88.7
-- server-02  disk_pct  91.3
```

## Filtering by One Array Using the Other

After zipping, you can filter based on conditions on any field in the tuple. This is how you filter one array's elements using conditions on a parallel array:

```sql
-- Find metric values where the metric name ends with '_pct' and value > 80
SELECT
    host,
    arrayFilter(m -> m.2 > 80.0, arrayZip(metric_names, metric_values)) AS high_metrics
FROM metric_snapshots;
-- server-01: []
-- server-02: [('mem_pct',88.7),('disk_pct',91.3)]
```

## Combining Timestamps with Event Data

Zip event arrays with their corresponding timestamps to create structured event tuples:

```sql
CREATE TABLE user_events
(
    user_id UInt32,
    event_times Array(DateTime),
    event_types Array(String)
) ENGINE = Memory;

INSERT INTO user_events VALUES
    (1,
     ['2024-01-01 10:00:00', '2024-01-01 10:05:00', '2024-01-01 10:12:00'],
     ['login', 'page_view', 'purchase']),
    (2,
     ['2024-01-02 09:00:00', '2024-01-02 09:30:00'],
     ['login', 'logout']);

-- Get structured (time, event_type) tuples per user
SELECT
    user_id,
    arrayZip(event_times, event_types) AS timed_events
FROM user_events;

-- Find only purchase events with their timestamps
SELECT
    user_id,
    arrayFilter(e -> e.2 = 'purchase', arrayZip(event_times, event_types)) AS purchases
FROM user_events;
```

## Transposing Row Arrays into Column Pairs

When multiple columns each contain array values and you want to analyze column relationships, zip brings them together:

```sql
CREATE TABLE ab_test_results
(
    experiment_id UInt32,
    variant_names Array(String),
    conversion_rates Array(Float32),
    sample_sizes Array(UInt32)
) ENGINE = Memory;

INSERT INTO ab_test_results VALUES
    (1, ['control', 'variant_a', 'variant_b'], [0.032, 0.041, 0.038], [5000, 4900, 5100]);

-- Zip all three arrays for a structured result
SELECT
    experiment_id,
    r.1 AS variant,
    r.2 AS conversion_rate,
    r.3 AS sample_size
FROM ab_test_results
ARRAY JOIN arrayZip(variant_names, conversion_rates, sample_sizes) AS r;
-- 1  control    0.032  5000
-- 1  variant_a  0.041  4900
-- 1  variant_b  0.038  5100
```

## Sorting One Array by Another Using Zip

Zip the target array with a sort key array, sort the tuples, then extract the reordered values:

```sql
-- Sort names by their corresponding scores (descending)
SELECT
    arrayMap(x -> x.1,
        arrayReverseSort(x -> x.2,
            arrayZip(['alice', 'bob', 'carol'], [88, 95, 72])
        )
    ) AS sorted_names;
-- Result: ['bob', 'alice', 'carol']  (sorted by score desc: 95, 88, 72)
```

## Summary

`arrayZip` is the ClickHouse way to merge parallel arrays into structured tuples for joint processing. It enables filtering, sorting, and unnesting of arrays that share a common index - such as metric names with values, timestamps with event types, or variant names with statistics. After zipping, tuple element access (`.1`, `.2`) and `arrayFilter` with lambda functions give you full control over the combined data.
