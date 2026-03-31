# How to Use indexOf() Function for Arrays in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Arrays, indexOf, Sql, Data Engineering

Description: Learn how to use indexOf() in ClickHouse to find the position of an element in an array, and how to handle cases where the element is not found.

---

## Overview

`indexOf(arr, elem)` returns the 1-based position of the first occurrence of `elem` in `arr`. If the element is not found, it returns 0. This function is useful for lookups, rank detection, and position-aware filtering.

## Basic Usage

```sql
SELECT indexOf(['a', 'b', 'c', 'd'], 'c') AS position
-- 3

SELECT indexOf(['a', 'b', 'c', 'd'], 'z') AS position
-- 0 (not found)
```

With numeric arrays:

```sql
SELECT indexOf([10, 20, 30, 40, 50], 30) AS position
-- 3
```

## Checking Membership vs. Position

`indexOf` returns the position, while `has()` returns 0 or 1. Use `indexOf > 0` as a membership test:

```sql
SELECT
    indexOf(['x', 'y', 'z'], 'y') > 0 AS found_with_indexof,
    has(['x', 'y', 'z'], 'y')         AS found_with_has
-- Both: 1
```

## Finding Priority Rank

A common use case is finding the rank of an item in a priority list:

```sql
SELECT
    incident_id,
    severity_levels,
    indexOf(severity_levels, 'critical') AS critical_position
FROM incidents
WHERE indexOf(severity_levels, 'critical') > 0
```

## Using indexOf with Tables

```sql
CREATE TABLE feature_flags
(
    user_id       UInt64,
    enabled_flags Array(String)
)
ENGINE = MergeTree()
ORDER BY user_id;

INSERT INTO feature_flags VALUES
(1, ['dark_mode', 'beta_dashboard', 'new_nav']),
(2, ['dark_mode']),
(3, ['beta_dashboard', 'new_nav', 'dark_mode']);
```

Find users who have `beta_dashboard` enabled and at what position:

```sql
SELECT
    user_id,
    indexOf(enabled_flags, 'beta_dashboard') AS flag_position
FROM feature_flags
WHERE indexOf(enabled_flags, 'beta_dashboard') > 0
```

## Extracting Elements by Found Position

Combine `indexOf` with `arrayElement` to extract the next element after a match:

```sql
SELECT
    arr,
    indexOf(arr, 'B')                             AS b_pos,
    arrayElement(arr, indexOf(arr, 'B') + 1)      AS element_after_b
FROM (SELECT ['A', 'B', 'C', 'D'] AS arr)
```

Output:

```text
arr              | b_pos | element_after_b
['A','B','C','D'] | 2     | C
```

## Finding First Occurrence in Repeated Values

`indexOf` always returns the first match:

```sql
SELECT indexOf([1, 2, 3, 2, 1], 2) AS first_2_position
-- 2
```

## Using indexOf in Sorting

Order results by the position of a priority element:

```sql
SELECT
    user_id,
    enabled_flags,
    indexOf(enabled_flags, 'dark_mode') AS dark_mode_position
FROM feature_flags
ORDER BY dark_mode_position ASC
```

Rows where `dark_mode` appears first have lower position values; rows where it is absent have position 0.

## indexOf vs. arrayFirstIndex

`arrayFirstIndex(func, arr)` is a more flexible alternative that accepts a lambda predicate:

```sql
SELECT arrayFirstIndex(x -> x > 3, [1, 2, 3, 4, 5]) AS first_gt_3
-- 4
```

Use `arrayFirstIndex` when the match condition is a predicate rather than an exact value.

## Summary

`indexOf(arr, elem)` returns the 1-based index of the first match in a ClickHouse array, or 0 if not found. It is useful for rank detection, priority-based ordering, and chained element access. For predicate-based position lookup, `arrayFirstIndex()` is the more flexible counterpart.
