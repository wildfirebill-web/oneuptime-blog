# How to Use greatest() and least() Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Math Function, Numeric, Comparison

Description: Learn how greatest() and least() compare multiple values row-wise in ClickHouse, with examples for capping values, column comparison, and computing ranges.

---

`greatest()` and `least()` are row-level comparison functions that return the maximum or minimum among a set of arguments. Unlike the aggregate functions `MAX()` and `MIN()`, which operate across rows in a group, `greatest()` and `least()` operate across columns or expressions within a single row. This distinction is important: use `greatest()` when you want to compare two or more columns side-by-side for each row, not to find the maximum across all rows.

## Function Signatures

```text
greatest(val1, val2 [, val3, ...])   -- returns the largest of the arguments
least(val1, val2 [, val3, ...])      -- returns the smallest of the arguments
```

Both accept any comparable types (numeric, String, Date, DateTime). All arguments should be of compatible types.

## Basic Usage

```sql
SELECT
    greatest(3, 7, 2, 9, 1)   AS greatest_of_5,
    least(3, 7, 2, 9, 1)      AS least_of_5,
    greatest(-5, 0)            AS clamp_at_zero,
    least(42, 100)             AS cap_at_100;
```

## Setting Up Sample Data

Create a table recording scores from multiple assessment sources for each student.

```sql
CREATE TABLE student_assessments
(
    student_id  UInt64,
    name        String,
    quiz_score  Float64,
    exam_score  Float64,
    project_score Float64,
    extra_credit  Float64
)
ENGINE = MergeTree
ORDER BY student_id;

INSERT INTO student_assessments VALUES
(1, 'Alice',  82.0, 91.0, 88.0,  5.0),
(2, 'Bob',    65.0, 58.0, 72.0,  0.0),
(3, 'Carol',  95.0, 97.0, 93.0, 10.0),
(4, 'Dave',   40.0, 55.0, 48.0,  2.0),
(5, 'Eve',    78.0, 78.0, 78.0,  0.0);
```

## Finding Best and Worst Score Per Student

Use `greatest()` to find each student's highest individual score, and `least()` to find their lowest.

```sql
SELECT
    name,
    quiz_score,
    exam_score,
    project_score,
    greatest(quiz_score, exam_score, project_score) AS best_score,
    least(quiz_score, exam_score, project_score)    AS worst_score,
    greatest(quiz_score, exam_score, project_score) -
    least(quiz_score, exam_score, project_score)    AS score_range
FROM student_assessments
ORDER BY best_score DESC;
```

## Clamping Values to a Range

Combine `greatest()` and `least()` to clamp a value between a minimum and maximum bound. This pattern is equivalent to `CLAMP(x, lo, hi)` in other languages.

```sql
SELECT
    student_id,
    name,
    quiz_score + extra_credit                                         AS raw_final,
    least(greatest(quiz_score + extra_credit, 0.0), 100.0)           AS clamped_final
FROM student_assessments;
```

The inner `greatest(..., 0)` prevents negative scores, and the outer `least(..., 100)` caps at 100.

## Setting a Minimum Floor on Values

Use `greatest()` with a constant to ensure a value never falls below a floor. This is useful for minimum price guarantees, minimum pay rates, or minimum metric thresholds.

```sql
CREATE TABLE pricing
(
    product_id   UInt64,
    base_price   Float64,
    discount_pct Float64,
    min_price    Float64
)
ENGINE = MergeTree
ORDER BY product_id;

INSERT INTO pricing VALUES
(1, 100.0, 0.20, 70.0),
(2,  50.0, 0.15, 40.0),
(3, 200.0, 0.50, 80.0),
(4,  25.0, 0.10, 20.0);
```

```sql
SELECT
    product_id,
    base_price,
    round(base_price * (1 - discount_pct), 2)                          AS discounted,
    greatest(base_price * (1 - discount_pct), min_price)               AS final_price
FROM pricing;
```

## Comparing Across Date Columns

`greatest()` and `least()` also work on Date and DateTime columns, returning the later or earlier of two dates.

```sql
CREATE TABLE project_dates
(
    project_id      UInt64,
    planned_start   Date,
    actual_start    Date,
    planned_end     Date,
    actual_end      Date
)
ENGINE = MergeTree
ORDER BY project_id;

INSERT INTO project_dates VALUES
(1, '2024-01-01', '2024-01-03', '2024-03-31', '2024-04-10'),
(2, '2024-02-01', '2024-01-28', '2024-04-30', '2024-04-25'),
(3, '2024-03-01', '2024-03-01', '2024-05-31', '2024-06-15');
```

```sql
SELECT
    project_id,
    greatest(planned_start, actual_start)  AS later_start,
    least(planned_end, actual_end)         AS earlier_completion,
    actual_end > planned_end               AS was_late
FROM project_dates;
```

## Summary

`greatest()` and `least()` perform row-wise comparisons across multiple column or expression arguments in ClickHouse. Use `greatest()` to find the maximum value across several columns, enforce a floor value, or find the latest of multiple dates. Use `least()` to find the minimum across columns, cap values at a ceiling, or find the earliest date. Combine both functions to clamp values within a defined range: `least(greatest(x, lo), hi)`. These functions are distinct from the aggregate functions MAX() and MIN(), which aggregate across rows - `greatest()` and `least()` operate within a single row.
