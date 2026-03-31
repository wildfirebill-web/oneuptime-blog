# How to Use DATEDIFF() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Date Function, Database, Query

Description: Learn how to use MySQL DATEDIFF() to calculate the number of days between two dates, with practical examples for age, deadlines, and time-series queries.

---

## What DATEDIFF() Does

`DATEDIFF()` returns the number of days between two date expressions. It subtracts the second argument from the first and returns an integer - positive when the first date is later, negative when it is earlier, and zero when they are the same day.

```sql
SELECT DATEDIFF('2026-03-31', '2026-03-01');
-- Result: 30
```

The function signature is:

```sql
DATEDIFF(expr1, expr2)
```

Both arguments can be `DATE`, `DATETIME`, or `TIMESTAMP` values. When `DATETIME` values are used, only the date portion is considered - the time component is ignored.

## Basic Examples

```sql
-- Days since a past event
SELECT DATEDIFF(CURDATE(), '2026-01-01') AS days_into_year;

-- Days until a future deadline
SELECT DATEDIFF('2026-12-31', CURDATE()) AS days_remaining;

-- Negative result when first date is earlier
SELECT DATEDIFF('2026-01-01', '2026-03-31') AS result;
-- -89
```

## Using DATEDIFF with a Table

Consider a `projects` table:

```sql
CREATE TABLE projects (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  start_date DATE NOT NULL,
  due_date DATE NOT NULL,
  completed_date DATE
);

INSERT INTO projects (name, start_date, due_date, completed_date) VALUES
  ('Alpha', '2026-01-01', '2026-02-01', '2026-01-28'),
  ('Beta',  '2026-02-01', '2026-03-01', '2026-03-05'),
  ('Gamma', '2026-03-01', '2026-04-01', NULL);
```

Calculate project duration and whether it finished on time:

```sql
SELECT
  name,
  DATEDIFF(due_date, start_date)       AS allotted_days,
  DATEDIFF(completed_date, start_date) AS actual_days,
  DATEDIFF(completed_date, due_date)   AS days_over_due
FROM projects
WHERE completed_date IS NOT NULL;
```

```text
Alpha  31  27  -4
Beta   28  32   4
```

## Finding Overdue or Upcoming Records

Identify tasks overdue by more than 7 days:

```sql
SELECT name, due_date, DATEDIFF(CURDATE(), due_date) AS days_overdue
FROM projects
WHERE completed_date IS NULL
  AND DATEDIFF(CURDATE(), due_date) > 7;
```

Find tasks due within the next 3 days:

```sql
SELECT name, due_date
FROM projects
WHERE completed_date IS NULL
  AND DATEDIFF(due_date, CURDATE()) BETWEEN 0 AND 3;
```

## DATEDIFF in Aggregations

Count how many days on average a project runs late:

```sql
SELECT
  AVG(DATEDIFF(completed_date, due_date)) AS avg_days_late
FROM projects
WHERE completed_date IS NOT NULL
  AND completed_date > due_date;
```

Group projects by how many weeks they took:

```sql
SELECT
  FLOOR(DATEDIFF(completed_date, start_date) / 7) AS weeks_taken,
  COUNT(*) AS project_count
FROM projects
WHERE completed_date IS NOT NULL
GROUP BY weeks_taken
ORDER BY weeks_taken;
```

## DATEDIFF vs TIMESTAMPDIFF

`DATEDIFF()` always returns a result in days. If you need the difference in other units (hours, months, years), use `TIMESTAMPDIFF()`:

```sql
-- Difference in months
SELECT TIMESTAMPDIFF(MONTH, '2026-01-15', '2026-03-31');
-- 2

-- Difference in hours
SELECT TIMESTAMPDIFF(HOUR, '2026-03-31 08:00:00', '2026-03-31 14:30:00');
-- 6
```

## Handling NULLs

If either argument is `NULL`, `DATEDIFF()` returns `NULL`:

```sql
SELECT DATEDIFF(NULL, '2026-01-01');
-- NULL

SELECT DATEDIFF(CURDATE(), NULL);
-- NULL
```

Use `COALESCE()` to substitute a default when a date might be `NULL`:

```sql
SELECT
  name,
  DATEDIFF(COALESCE(completed_date, CURDATE()), start_date) AS elapsed_days
FROM projects;
```

## Summary

`DATEDIFF(expr1, expr2)` computes the integer number of days between two dates, ignoring the time portion. Use it for deadline tracking, duration calculations, and time-series filtering. For differences in other time units, use `TIMESTAMPDIFF()`. Always handle potential `NULL` values with `COALESCE()` to avoid unexpected null results in your queries.
