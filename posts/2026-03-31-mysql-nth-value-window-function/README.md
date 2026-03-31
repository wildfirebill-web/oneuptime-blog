# How to Use NTH_VALUE() Window Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Window Function, Ranking

Description: Learn how to use MySQL's NTH_VALUE() window function to retrieve the Nth value in a window frame, with examples for comparisons and running analysis.

---

## What Is NTH_VALUE()?

`NTH_VALUE()` is a window function introduced in MySQL 8.0 that returns the value of the expression from the Nth row of the window frame. It is a generalization of `FIRST_VALUE()` and `LAST_VALUE()` - `FIRST_VALUE()` is equivalent to `NTH_VALUE(expr, 1)`.

Syntax:

```sql
NTH_VALUE(expr, N) OVER (
  [PARTITION BY partition_expr]
  ORDER BY sort_expr
  [frame_clause]
)
```

`N` must be a positive integer. Returns `NULL` if the Nth row does not exist in the window.

## Basic Example

```sql
CREATE TABLE race_results (
  race_id INT,
  runner VARCHAR(50),
  finish_time DECIMAL(6,2)
);

INSERT INTO race_results VALUES
  (1, 'Alice', 12.50),
  (1, 'Bob', 13.20),
  (1, 'Carol', 11.80),
  (1, 'Dave', 14.10),
  (1, 'Eve', 12.90);

SELECT
  runner,
  finish_time,
  NTH_VALUE(finish_time, 1) OVER (ORDER BY finish_time) AS first_place_time,
  NTH_VALUE(finish_time, 2) OVER (ORDER BY finish_time) AS second_place_time,
  NTH_VALUE(finish_time, 3) OVER (ORDER BY finish_time) AS third_place_time
FROM race_results
WHERE race_id = 1;
```

## The Frame Clause Matters

By default, the window frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. For `NTH_VALUE()` to see all rows in the partition (not just up to the current row), use `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`:

```sql
SELECT
  runner,
  finish_time,
  NTH_VALUE(finish_time, 2) OVER (
    ORDER BY finish_time
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS second_fastest
FROM race_results
WHERE race_id = 1;
```

Without specifying the full frame, rows earlier in the order may see `NULL` for the 2nd value before the 2nd row has been reached.

## Using with PARTITION BY

Find the 2nd fastest time per race:

```sql
INSERT INTO race_results VALUES
  (2, 'Frank', 11.50), (2, 'Grace', 10.90), (2, 'Heidi', 12.30);

SELECT
  race_id,
  runner,
  finish_time,
  NTH_VALUE(finish_time, 2) OVER (
    PARTITION BY race_id
    ORDER BY finish_time
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS second_place
FROM race_results
ORDER BY race_id, finish_time;
```

## Comparing Each Row to the Nth Best

Compute the gap between each runner's time and the 2nd place time:

```sql
SELECT
  runner,
  finish_time,
  NTH_VALUE(finish_time, 2) OVER w AS second_time,
  ROUND(
    finish_time - NTH_VALUE(finish_time, 2) OVER w, 2
  ) AS gap_to_second
FROM race_results
WHERE race_id = 1
WINDOW w AS (
  ORDER BY finish_time
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
ORDER BY finish_time;
```

## NULL Returns for Out-of-Range N

```sql
SELECT
  runner,
  NTH_VALUE(runner, 10) OVER (ORDER BY finish_time) AS tenth_runner
FROM race_results WHERE race_id = 1;
-- Returns NULL for all rows (only 5 runners)
```

## Named Window Reuse

Define the window once and reference it multiple times:

```sql
SELECT
  runner,
  finish_time,
  NTH_VALUE(finish_time, 1) OVER w AS gold,
  NTH_VALUE(finish_time, 2) OVER w AS silver,
  NTH_VALUE(finish_time, 3) OVER w AS bronze
FROM race_results
WHERE race_id = 1
WINDOW w AS (
  ORDER BY finish_time
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
ORDER BY finish_time;
```

## Summary

`NTH_VALUE()` returns the expression value from the Nth row of the window frame. Always specify `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` when you need every row in the result to see all partition values. Use named windows to avoid repeating the frame clause when calling `NTH_VALUE()` multiple times.
