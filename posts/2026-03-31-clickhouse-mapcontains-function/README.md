# How to Use mapContains() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Map Function, Filtering, Query Optimization

Description: Learn how mapContains() checks for key presence in ClickHouse Map columns, with examples for filtering rows, validating map contents, and feature flag checks.

---

When working with Map columns in ClickHouse, a common requirement is to filter rows based on whether a particular key is present in the map - regardless of its value. The `mapContains()` function provides this capability efficiently. Unlike bracket access (`map['key']`), which returns a default value when the key is absent, `mapContains()` clearly distinguishes between "key is present with a zero/empty value" and "key is not present at all."

## Function Signature

```text
mapContains(map, key) -> UInt8
```

Returns `1` if the map contains the specified key, `0` otherwise. The key argument must match the key type of the map.

## Basic Usage

Verify the function behavior with inline map literals before applying it to table data.

```sql
SELECT
    mapContains(map('a', 1, 'b', 2), 'a') AS has_a,
    mapContains(map('a', 1, 'b', 2), 'c') AS has_c;
```

This returns `1` for `has_a` and `0` for `has_c`.

## Setting Up a Sample Table

Create a table that stores per-request feature flags as a Map. This is a realistic pattern for A/B testing or gradual feature rollout systems.

```sql
CREATE TABLE request_context
(
    request_id  UUID,
    user_id     UInt64,
    occurred_at DateTime,
    flags       Map(String, String)
)
ENGINE = MergeTree
ORDER BY (user_id, occurred_at);

INSERT INTO request_context VALUES
('a1b2c3d4-0000-0000-0000-000000000001', 101, '2024-02-01 09:00:00', map('experiment_v2', 'enabled', 'dark_mode', 'on')),
('a1b2c3d4-0000-0000-0000-000000000002', 102, '2024-02-01 09:05:00', map('dark_mode', 'off')),
('a1b2c3d4-0000-0000-0000-000000000003', 103, '2024-02-01 09:10:00', map('experiment_v2', 'control', 'new_nav', 'enabled')),
('a1b2c3d4-0000-0000-0000-000000000004', 104, '2024-02-01 09:15:00', map('new_nav', 'disabled')),
('a1b2c3d4-0000-0000-0000-000000000005', 105, '2024-02-01 09:20:00', map('experiment_v2', 'enabled', 'new_nav', 'enabled', 'dark_mode', 'on'));
```

## Filtering Rows by Key Presence

Use `mapContains()` in a WHERE clause to return only rows that include a specific key. This is more reliable than checking `flags['key'] != ''` because it correctly handles keys with empty string values.

```sql
SELECT
    request_id,
    user_id,
    flags['experiment_v2'] AS experiment_group
FROM request_context
WHERE mapContains(flags, 'experiment_v2') = 1;
```

## Counting Rows with and Without a Key

Aggregate using `mapContains()` to measure coverage - for example, how many requests include a given flag.

```sql
SELECT
    countIf(mapContains(flags, 'experiment_v2') = 1)  AS in_experiment,
    countIf(mapContains(flags, 'experiment_v2') = 0)  AS not_in_experiment,
    count()                                            AS total_requests
FROM request_context;
```

## Checking Multiple Keys at Once

Combine multiple `mapContains()` calls with AND or OR logic to filter by the presence of several keys simultaneously.

```sql
SELECT
    user_id,
    flags
FROM request_context
WHERE
    mapContains(flags, 'experiment_v2') = 1
    AND mapContains(flags, 'new_nav') = 1;
```

This returns only requests that were part of both the experiment and had the new navigation flag set.

## Using mapContains() in a CASE Expression

Classify rows based on which keys are present, creating a derived category column from map contents.

```sql
SELECT
    request_id,
    user_id,
    CASE
        WHEN mapContains(flags, 'experiment_v2') = 1 AND mapContains(flags, 'new_nav') = 1
            THEN 'full_experiment'
        WHEN mapContains(flags, 'experiment_v2') = 1
            THEN 'experiment_only'
        WHEN mapContains(flags, 'new_nav') = 1
            THEN 'new_nav_only'
        ELSE 'control'
    END AS user_segment
FROM request_context;
```

## Validating Required Keys Before Access

When a map is expected to always contain certain keys, use `mapContains()` in a guard to avoid processing rows with missing required fields.

```sql
SELECT
    request_id,
    flags['experiment_v2'] AS experiment_value
FROM request_context
WHERE mapContains(flags, 'experiment_v2') = 1
  AND flags['experiment_v2'] = 'enabled';
```

## Building a Key Coverage Report

Use `mapContains()` across multiple keys to produce a coverage matrix showing which flags are present in what percentage of rows.

```sql
SELECT
    round(100 * avg(mapContains(flags, 'experiment_v2')), 1) AS pct_experiment_v2,
    round(100 * avg(mapContains(flags, 'dark_mode')), 1)     AS pct_dark_mode,
    round(100 * avg(mapContains(flags, 'new_nav')), 1)       AS pct_new_nav
FROM request_context;
```

## Summary

`mapContains()` is the correct tool for checking key presence in ClickHouse Map columns. It returns `1` when a key exists and `0` when it does not, making it safe to use even when keys may have empty or zero values. Use it in WHERE clauses for filtering, in CASE expressions for classification, in aggregate functions for coverage analysis, and as a guard before accessing map values that may not always be present. It works with any Map key type including String, Integer, and other comparable types.
