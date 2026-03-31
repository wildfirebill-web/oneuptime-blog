# How to Use gcd() and lcm() Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Math Function, Integer, Numeric

Description: Learn how gcd() computes the greatest common divisor and lcm() computes the least common multiple in ClickHouse, with examples for fraction simplification and scheduling.

---

`gcd()` and `lcm()` are integer number theory functions in ClickHouse. `gcd(a, b)` returns the greatest common divisor - the largest positive integer that divides both a and b without remainder. `lcm(a, b)` returns the least common multiple - the smallest positive integer divisible by both a and b. These functions appear in fraction reduction, period alignment for cyclic processes, scheduling calculations, and any domain where divisibility relationships are meaningful.

## Function Signatures

```text
gcd(a, b)   -- greatest common divisor of a and b
lcm(a, b)   -- least common multiple of a and b
```

Both accept integer types (UInt8 through UInt64, Int8 through Int64) and return the same type as the input. For signed types, the result is always non-negative.

## Basic Usage

```sql
SELECT
    gcd(12, 8)    AS gcd_12_8,
    gcd(100, 75)  AS gcd_100_75,
    gcd(17, 5)    AS gcd_coprime,
    lcm(4, 6)     AS lcm_4_6,
    lcm(12, 8)    AS lcm_12_8,
    lcm(5, 7)     AS lcm_coprime;
```

`gcd(12, 8)` = 4, `gcd(17, 5)` = 1 (coprime). `lcm(4, 6)` = 12, `lcm(5, 7)` = 35.

The relationship between GCD and LCM is: `lcm(a, b) = a * b / gcd(a, b)`.

## Fraction Simplification

Use `gcd()` to reduce fractions to their lowest terms by dividing numerator and denominator by their GCD.

```sql
SELECT
    numerator,
    denominator,
    gcd(numerator, denominator)            AS common_factor,
    numerator / gcd(numerator, denominator) AS reduced_numerator,
    denominator / gcd(numerator, denominator) AS reduced_denominator
FROM (
    SELECT
        arrayJoin([12, 15, 8, 100, 36]) AS numerator,
        arrayJoin([18, 25, 12, 75,  48]) AS denominator
);
```

## Detecting Coprime Pairs

Two numbers are coprime if their GCD is 1. Use `gcd()` to filter or flag coprime pairs.

```sql
SELECT
    a,
    b,
    gcd(a, b) AS g,
    gcd(a, b) = 1 AS are_coprime
FROM (
    SELECT
        arrayJoin([8, 9, 15, 14]) AS a,
        arrayJoin([9, 6, 14, 21]) AS b
);
```

## Period Alignment for Cyclic Processes

Use `lcm()` to find when two cyclic processes next synchronize. If process A repeats every `p` steps and process B every `q` steps, they align again at step `lcm(p, q)`.

```sql
CREATE TABLE batch_jobs
(
    job_name     String,
    period_hours UInt32
)
ENGINE = MergeTree
ORDER BY job_name;

INSERT INTO batch_jobs VALUES
('data_sync',   6),
('model_train', 8),
('report_gen',  12),
('backup',      24);
```

Find when each pair of jobs will next run at the same time.

```sql
SELECT
    a.job_name AS job_a,
    b.job_name AS job_b,
    a.period_hours AS period_a,
    b.period_hours AS period_b,
    lcm(a.period_hours, b.period_hours) AS next_sync_hours
FROM batch_jobs a
CROSS JOIN batch_jobs b
WHERE a.job_name < b.job_name
ORDER BY next_sync_hours;
```

## Scheduling Maintenance Windows

Determine when multiple system components requiring different maintenance cycles will all be available at the same time.

```sql
SELECT
    lcm(lcm(4, 6), 9) AS all_three_align_at_hours;
```

If component A needs maintenance every 4 hours, B every 6, and C every 9, they all align at hour `lcm(4, 6, 9)` = 36.

## Using GCD for Resolution Normalization

In display or imaging contexts, reduce a resolution ratio to its simplest form.

```sql
SELECT
    width,
    height,
    gcd(width, height)               AS common,
    width / gcd(width, height)       AS ratio_w,
    height / gcd(width, height)      AS ratio_h,
    concat(
        toString(width / gcd(width, height)),
        ':',
        toString(height / gcd(width, height))
    ) AS aspect_ratio
FROM (
    SELECT
        arrayJoin([1920, 1280, 2560, 3840]) AS width,
        arrayJoin([1080,  720, 1440, 2160]) AS height
);
```

## Summary

`gcd()` and `lcm()` provide number-theoretic functions for integer divisibility in ClickHouse. Use `gcd()` for fraction reduction, coprimality testing, and finding common measurement units. Use `lcm()` for period alignment of cyclic processes, scheduling coordination, and finding the smallest common interval. Both functions accept any integer type and return non-negative results. They compose naturally with arithmetic operators and are useful anywhere divisibility, repetition cycles, or rational number normalization arise in analytical queries.
