# How to Use modulo and moduloOrZero in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Math Function, Integer, Arithmetic

Description: Learn how modulo and moduloOrZero compute remainders in ClickHouse, with examples for cycle detection, round-robin assignment, and time-of-day extraction.

---

The modulo operation returns the remainder after integer division. In ClickHouse, you can use the `%` operator or the explicit `modulo(a, b)` function. `moduloOrZero(a, b)` provides a safe variant that returns 0 instead of throwing an exception when the divisor is zero. Modulo is fundamental for cyclic patterns: determining whether a number is even or odd, assigning items in round-robin fashion, extracting time components from Unix timestamps, and detecting periodicity in sequences.

## Syntax Options

```text
a % b              -- modulo operator
modulo(a, b)       -- function form, equivalent to a % b
moduloOrZero(a, b) -- returns 0 when b = 0
```

The result has the same sign as the dividend (ClickHouse uses truncated division semantics). For example, `-7 % 3` = -1 in ClickHouse.

## Basic Usage

```sql
SELECT
    10 % 3               AS mod_10_3,
    modulo(10, 3)        AS mod_function,
    7 % 2                AS odd_check,
    8 % 2                AS even_check,
    modulo(-7, 3)        AS neg_dividend,
    moduloOrZero(5, 0)   AS div_zero_safe;
```

`10 % 3` = 1, `7 % 2` = 1 (odd), `8 % 2` = 0 (even), `moduloOrZero(5, 0)` = 0.

## Even and Odd Detection

Use `% 2` to classify rows as even or odd, useful for alternating row styling, split-testing assignment, or interleaved data processing.

```sql
SELECT
    number,
    number % 2 = 0 AS is_even,
    CASE number % 2 WHEN 0 THEN 'even' ELSE 'odd' END AS parity
FROM (
    SELECT arrayJoin(range(1, 11)) AS number
);
```

## Round-Robin Worker Assignment

Assign tasks to workers evenly using modulo on the task ID.

```sql
CREATE TABLE task_queue
(
    task_id   UInt64,
    task_name String,
    priority  UInt8
)
ENGINE = MergeTree
ORDER BY task_id;

INSERT INTO task_queue SELECT
    number + 1,
    concat('task_', toString(number + 1)),
    (number % 3) + 1
FROM numbers(15);
```

```sql
SELECT
    task_id,
    task_name,
    (task_id - 1) % 4 AS assigned_worker
FROM task_queue
ORDER BY assigned_worker, task_id;
```

## Time-of-Day Extraction from Unix Timestamp

Extract hours, minutes, and seconds from a Unix timestamp using modulo arithmetic.

```sql
CREATE TABLE raw_events
(
    event_id   UInt64,
    unix_ts    UInt64
)
ENGINE = MergeTree
ORDER BY event_id;

INSERT INTO raw_events VALUES
(1, 1704067200),
(2, 1704078900),
(3, 1704110461),
(4, 1704153601);
```

```sql
SELECT
    event_id,
    unix_ts,
    toDateTime(unix_ts)                              AS human_time,
    intDiv(unix_ts % 86400, 3600)                    AS hour_of_day,
    intDiv(unix_ts % 3600, 60)                       AS minute_of_hour,
    unix_ts % 60                                     AS second_of_minute
FROM raw_events;
```

## Detecting Periodic Patterns

Check whether events occur on a regular period by using modulo to test divisibility.

```sql
CREATE TABLE sequence_events
(
    seq_num    UInt64,
    value      Float64
)
ENGINE = MergeTree
ORDER BY seq_num;

INSERT INTO sequence_events SELECT number + 1, sin(number * 0.5) FROM numbers(50);
```

Find rows that fall on every 7th position (weekly cadence if seq_num represents days).

```sql
SELECT
    seq_num,
    value,
    seq_num % 7 AS day_of_week
FROM sequence_events
WHERE seq_num % 7 = 0
ORDER BY seq_num;
```

## Cyclic Histogram Bucketing

Use modulo to fold values into a cyclic range, such as mapping any hour value modulo 24 to ensure it falls within 0-23.

```sql
SELECT
    raw_hour,
    raw_hour % 24 AS normalized_hour
FROM (
    SELECT arrayJoin([0, 6, 12, 18, 24, 30, 48]) AS raw_hour
);
```

## Safe Modulo for Conditional Data

When the divisor comes from a data column and might be zero, use `moduloOrZero()` to prevent failures.

```sql
CREATE TABLE divisor_data
(
    id        UInt64,
    value     UInt64,
    group_size UInt64
)
ENGINE = MergeTree
ORDER BY id;

INSERT INTO divisor_data VALUES
(1, 100, 7),
(2, 200, 0),
(3, 150, 5);
```

```sql
SELECT
    id,
    value,
    group_size,
    moduloOrZero(value, group_size) AS remainder
FROM divisor_data;
```

## Summary

Modulo in ClickHouse is available as the `%` operator, the `modulo()` function, and the `moduloOrZero()` variant for safe zero-divisor handling. Use it for even/odd classification, round-robin distribution, Unix timestamp decomposition, cycle detection, and folding values into a periodic range. The result follows truncated-division sign rules (same sign as the dividend). Combine modulo with `intDiv()` to extract both quotient and remainder in the same expression, which is the basis for extracting time components from raw Unix timestamps.
