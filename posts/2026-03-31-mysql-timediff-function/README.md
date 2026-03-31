# How to Use TIMEDIFF() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Date Function, Time Function, Database

Description: Learn how to use MySQL TIMEDIFF() to calculate the difference between two time or datetime values, returning the result as a TIME expression.

---

## What TIMEDIFF() Does

`TIMEDIFF()` subtracts the second time expression from the first and returns the result as a `TIME` value. Unlike `DATEDIFF()`, which returns an integer number of days, `TIMEDIFF()` preserves hours, minutes, and seconds, including the sign.

```sql
SELECT TIMEDIFF('14:30:00', '09:00:00');
-- Result: 05:30:00
```

The function signature is:

```sql
TIMEDIFF(expr1, expr2)
```

Both arguments must be the same type - either both `TIME` or both `DATETIME`/`TIMESTAMP`. Mixing types (e.g., `TIME` with `DATETIME`) produces `NULL`.

## Basic Examples

```sql
-- Positive difference
SELECT TIMEDIFF('18:00:00', '09:30:00');
-- 08:30:00

-- Negative difference (second time is later)
SELECT TIMEDIFF('09:00:00', '18:00:00');
-- -09:00:00

-- With DATETIME values (date portion matters)
SELECT TIMEDIFF('2026-03-31 14:00:00', '2026-03-30 14:00:00');
-- 24:00:00
```

## Using TIMEDIFF with a Table

Consider a `work_sessions` table tracking employee shifts:

```sql
CREATE TABLE work_sessions (
  id INT AUTO_INCREMENT PRIMARY KEY,
  employee VARCHAR(100),
  clock_in  DATETIME NOT NULL,
  clock_out DATETIME
);

INSERT INTO work_sessions (employee, clock_in, clock_out) VALUES
  ('Alice', '2026-03-31 08:00:00', '2026-03-31 17:15:00'),
  ('Bob',   '2026-03-31 09:30:00', '2026-03-31 18:00:00'),
  ('Carol', '2026-03-31 07:45:00', NULL);
```

Calculate hours worked per session:

```sql
SELECT
  employee,
  TIMEDIFF(clock_out, clock_in) AS hours_worked
FROM work_sessions
WHERE clock_out IS NOT NULL;
```

```text
Alice  09:15:00
Bob    08:30:00
```

## Converting TIMEDIFF Results to Numeric Values

`TIMEDIFF()` returns a `TIME` value, not a number. To get total minutes or hours as a number, wrap the result with `TIME_TO_SEC()`:

```sql
-- Total seconds
SELECT TIME_TO_SEC(TIMEDIFF('17:15:00', '08:00:00')) AS total_seconds;
-- 33300

-- Total hours (decimal)
SELECT TIME_TO_SEC(TIMEDIFF(clock_out, clock_in)) / 3600 AS hours_decimal
FROM work_sessions
WHERE clock_out IS NOT NULL;
```

## Filtering by Duration

Find sessions longer than 8 hours:

```sql
SELECT employee, clock_in, clock_out,
       TIMEDIFF(clock_out, clock_in) AS duration
FROM work_sessions
WHERE clock_out IS NOT NULL
  AND TIMEDIFF(clock_out, clock_in) > '08:00:00';
```

Find sessions shorter than 6 hours:

```sql
SELECT employee,
       TIMEDIFF(clock_out, clock_in) AS duration
FROM work_sessions
WHERE clock_out IS NOT NULL
  AND TIME_TO_SEC(TIMEDIFF(clock_out, clock_in)) < 21600;
```

## Handling NULLs and Invalid Inputs

If either argument is `NULL`, `TIMEDIFF()` returns `NULL`:

```sql
SELECT TIMEDIFF(NULL, '09:00:00');
-- NULL
```

Use `COALESCE()` to substitute the current time for ongoing sessions:

```sql
SELECT
  employee,
  TIMEDIFF(COALESCE(clock_out, NOW()), clock_in) AS elapsed
FROM work_sessions;
```

If you pass mismatched types (one `TIME` and one `DATETIME`), MySQL returns `NULL` with a warning. Always ensure both arguments share the same type.

## TIMEDIFF vs TIMESTAMPDIFF

`TIMEDIFF()` returns a `TIME` value and works well when you need `HH:MM:SS` output or want to compare results with `TIME` constants. `TIMESTAMPDIFF()` returns an integer in a specified unit and is better when you need to count hours, minutes, or days as plain numbers:

```sql
-- TIMEDIFF: returns 09:15:00
SELECT TIMEDIFF('2026-03-31 17:15:00', '2026-03-31 08:00:00');

-- TIMESTAMPDIFF: returns 9 (integer hours)
SELECT TIMESTAMPDIFF(HOUR, '2026-03-31 08:00:00', '2026-03-31 17:15:00');
```

## Summary

`TIMEDIFF(expr1, expr2)` returns the difference between two time or datetime values as a signed `TIME` result. Use `TIME_TO_SEC()` to convert it to a numeric value for arithmetic. For unit-specific integer differences, prefer `TIMESTAMPDIFF()`. Always handle `NULL` values with `COALESCE()` for ongoing or incomplete records.
