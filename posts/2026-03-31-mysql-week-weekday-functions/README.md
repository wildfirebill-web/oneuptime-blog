# How to Use WEEK() and WEEKDAY() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Date Function, Database

Description: Learn how to use MySQL WEEK() to get the week number of a date and WEEKDAY() to get the day of the week index with Monday-based numbering.

---

## Overview

MySQL provides two functions for week-related calculations:

- `WEEK()` - returns the calendar week number (0 or 1 through 53) for a date.
- `WEEKDAY()` - returns the weekday index with Monday = 0 through Sunday = 6.

Both are commonly used for weekly reporting, scheduling, and ISO week-based grouping.

---

## WEEK() Function

Returns the week number for a given date.

**Syntax:**

```sql
WEEK(date[, mode])
```

- `date` - a `DATE` or `DATETIME` value.
- `mode` - an optional integer (0-7) that controls the week numbering convention.
- Returns `NULL` if `date` is `NULL`.

### Default Behavior (mode 0)

```sql
SELECT WEEK('2026-03-31');
-- Returns: 13 (or similar, depending on year)

SELECT WEEK('2026-01-01');
-- Returns: 0 or 1 depending on which day the year starts
```

---

## WEEK() Mode Values

The `mode` argument controls how the first week of the year is determined:

| Mode | First Day of Week | Week 1 Definition                         |
|------|-------------------|-------------------------------------------|
| 0    | Sunday            | Week with the first Sunday of the year    |
| 1    | Monday            | Week with 4+ days in the new year         |
| 2    | Sunday            | Week with the first Sunday (week 1 start) |
| 3    | Monday            | ISO 8601 - week with first Thursday       |
| 4    | Sunday            | Week with 4+ days in the new year         |
| 5    | Monday            | Week with the first Monday of the year    |
| 6    | Sunday            | Week with 4+ days in the new year         |
| 7    | Monday            | Week with the first Monday (week 1 start) |

### Common Mode Examples

```sql
-- Mode 0: Sunday-based, range 0-53
SELECT WEEK('2026-01-01', 0);

-- Mode 1: Monday-based, ISO-like
SELECT WEEK('2026-01-01', 1);

-- Mode 3: ISO 8601 (same as WEEKOFYEAR())
SELECT WEEK('2026-01-01', 3);
SELECT WEEKOFYEAR('2026-01-01');  -- Equivalent to WEEK(date, 3)
```

---

## WEEKOFYEAR() - ISO Week Number

`WEEKOFYEAR()` is a convenience function equivalent to `WEEK(date, 3)`, returning ISO 8601 week numbers (1-53, always Monday-based):

```sql
SELECT WEEKOFYEAR('2026-03-31');
-- Returns: 14

SELECT WEEKOFYEAR('2026-01-01');
-- Returns: 1

SELECT WEEKOFYEAR('2025-12-29');
-- Returns: 1  (may belong to ISO week 1 of 2026)
```

---

## WEEKDAY() Function

Returns the weekday index for a date using Monday = 0 convention.

**Syntax:**

```sql
WEEKDAY(date)
```

| Return Value | Weekday   |
|--------------|-----------|
| 0            | Monday    |
| 1            | Tuesday   |
| 2            | Wednesday |
| 3            | Thursday  |
| 4            | Friday    |
| 5            | Saturday  |
| 6            | Sunday    |

```sql
SELECT WEEKDAY('2026-03-31');
-- Returns: 1  (Tuesday)

SELECT WEEKDAY('2026-03-30');
-- Returns: 0  (Monday)

SELECT WEEKDAY('2026-04-05');
-- Returns: 6  (Sunday)

SELECT WEEKDAY(NULL);
-- Returns: NULL
```

---

## WEEKDAY() vs DAYOFWEEK()

Both return a weekday number, but start from different reference points:

```mermaid
flowchart LR
    A[Same date] -->|WEEKDAY()| B[Monday=0, Sunday=6]
    A -->|DAYOFWEEK()| C[Sunday=1, Saturday=7]
```

```sql
SELECT
    WEEKDAY('2026-03-31')   AS weekday_val,   -- 1 (Tue, 0=Mon)
    DAYOFWEEK('2026-03-31') AS dayofweek_val; -- 3 (Tue, 1=Sun)
```

Use `WEEKDAY()` for ISO-style (Monday-first) logic, `DAYOFWEEK()` for ODBC-style (Sunday-first) logic.

---

## Grouping by Week Number

```sql
CREATE TABLE sales (
    id INT AUTO_INCREMENT PRIMARY KEY,
    sale_date DATETIME,
    amount DECIMAL(10, 2)
);

-- Weekly revenue report
SELECT
    YEAR(sale_date)        AS year,
    WEEK(sale_date, 3)     AS iso_week,
    SUM(amount)            AS revenue,
    COUNT(*)               AS transactions
FROM sales
WHERE YEAR(sale_date) = 2026
GROUP BY YEAR(sale_date), WEEK(sale_date, 3)
ORDER BY year, iso_week;
```

---

## Finding Weekends Using WEEKDAY()

```sql
-- Saturday (5) and Sunday (6) orders
SELECT id, sale_date, amount
FROM sales
WHERE WEEKDAY(sale_date) >= 5;

-- Business days only (Monday-Friday)
SELECT id, sale_date, amount
FROM sales
WHERE WEEKDAY(sale_date) <= 4;
```

---

## Start of Week Calculation

```sql
-- Calculate the Monday start of the current week
SELECT DATE_SUB(CURDATE(), INTERVAL WEEKDAY(CURDATE()) DAY) AS week_start;

-- Calculate the Sunday end of the current week
SELECT DATE_ADD(
    DATE_SUB(CURDATE(), INTERVAL WEEKDAY(CURDATE()) DAY),
    INTERVAL 6 DAY
) AS week_end;
```

---

## Filtering by Specific Week

```sql
-- All sales in ISO week 14 of 2026
SELECT * FROM sales
WHERE WEEKOFYEAR(sale_date) = 14
  AND YEAR(sale_date) = 2026;
```

---

## WEEK() Edge Cases at Year Boundaries

```sql
-- December dates may belong to week 1 of the following year (ISO)
SELECT WEEKOFYEAR('2025-12-31');  -- May return 1 (belongs to ISO week 1 of 2026)
SELECT YEAR('2025-12-31');        -- Returns 2025 (calendar year)

-- January dates may belong to week 52/53 of the previous year
SELECT WEEKOFYEAR('2026-01-01');  -- Check if this is week 1 or 53
```

Always use `YEARWEEK()` together to avoid year-boundary ambiguity:

```sql
SELECT YEARWEEK('2025-12-31', 3);  -- Returns 202601 (year 2026, week 1)
SELECT YEARWEEK('2026-03-31', 3);  -- Returns 202614
```

---

## Summary

`WEEK()` returns the week number for a date with a configurable mode to handle different week-start conventions, and `WEEKDAY()` returns the day of week index from 0 (Monday) to 6 (Sunday). Use `WEEKOFYEAR()` as a convenient ISO 8601 week number function. For year-boundary correctness in weekly reports, pair `WEEK()` with `YEARWEEK()` to avoid misclassifying December/January dates. Use `WEEKDAY()` for Monday-first business logic and `DAYOFWEEK()` for Sunday-first ODBC-style indexing.
