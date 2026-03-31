# How to Use DATE_ADD() and DATE_SUB() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Date Function, Database

Description: Learn how to use MySQL DATE_ADD() and DATE_SUB() to add or subtract time intervals from dates and datetimes for scheduling and reporting queries.

---

## Overview

`DATE_ADD()` and `DATE_SUB()` are MySQL functions for adding or subtracting a time interval from a `DATE`, `DATETIME`, or `TIMESTAMP` value. They are fundamental for date arithmetic, generating future/past dates, building date ranges, and scheduling logic.

---

## DATE_ADD() Function

**Syntax:**

```sql
DATE_ADD(date, INTERVAL expr unit)
```

- `date` - a date or datetime value.
- `expr` - the quantity to add (can be a positive or negative integer, or a string for compound units).
- `unit` - the time unit keyword.

### Basic Examples

```sql
-- Add 7 days
SELECT DATE_ADD('2026-03-31', INTERVAL 7 DAY);
-- Returns: '2026-04-07'

-- Add 3 months
SELECT DATE_ADD('2026-01-31', INTERVAL 3 MONTH);
-- Returns: '2026-04-30'  (clamped to last day of April)

-- Add 1 year
SELECT DATE_ADD('2026-03-31', INTERVAL 1 YEAR);
-- Returns: '2027-03-31'

-- Add 2 hours
SELECT DATE_ADD('2026-03-31 10:00:00', INTERVAL 2 HOUR);
-- Returns: '2026-03-31 12:00:00'

-- Add 30 minutes
SELECT DATE_ADD(NOW(), INTERVAL 30 MINUTE);

-- Add negative interval (equivalent to subtracting)
SELECT DATE_ADD('2026-03-31', INTERVAL -5 DAY);
-- Returns: '2026-03-26'
```

---

## DATE_SUB() Function

**Syntax:**

```sql
DATE_SUB(date, INTERVAL expr unit)
```

`DATE_SUB()` is identical to `DATE_ADD()` but subtracts the interval instead of adding it. `DATE_SUB(d, INTERVAL n UNIT)` is equivalent to `DATE_ADD(d, INTERVAL -n UNIT)`.

### Basic Examples

```sql
-- Subtract 7 days
SELECT DATE_SUB('2026-03-31', INTERVAL 7 DAY);
-- Returns: '2026-03-24'

-- Subtract 1 month
SELECT DATE_SUB('2026-03-31', INTERVAL 1 MONTH);
-- Returns: '2026-02-28'

-- Subtract 6 months
SELECT DATE_SUB('2026-03-31', INTERVAL 6 MONTH);
-- Returns: '2025-09-30'

-- Subtract 45 seconds
SELECT DATE_SUB(NOW(), INTERVAL 45 SECOND);
```

---

## Supported INTERVAL Units

| Unit          | Example Expression       |
|---------------|--------------------------|
| `MICROSECOND` | `INTERVAL 500 MICROSECOND` |
| `SECOND`      | `INTERVAL 30 SECOND`     |
| `MINUTE`      | `INTERVAL 15 MINUTE`     |
| `HOUR`        | `INTERVAL 2 HOUR`        |
| `DAY`         | `INTERVAL 7 DAY`         |
| `WEEK`        | `INTERVAL 2 WEEK`        |
| `MONTH`       | `INTERVAL 3 MONTH`       |
| `QUARTER`     | `INTERVAL 1 QUARTER`     |
| `YEAR`        | `INTERVAL 1 YEAR`        |
| `SECOND_MICROSECOND` | `INTERVAL '30.5' SECOND_MICROSECOND` |
| `MINUTE_SECOND` | `INTERVAL '5:30' MINUTE_SECOND` |
| `HOUR_MINUTE` | `INTERVAL '1:30' HOUR_MINUTE` |
| `DAY_HOUR`    | `INTERVAL '2 12' DAY_HOUR` |
| `YEAR_MONTH`  | `INTERVAL '1-3' YEAR_MONTH` |

---

## Compound Interval Examples

```sql
-- Add 2 hours and 30 minutes
SELECT DATE_ADD('2026-03-31 09:00:00', INTERVAL '2:30' HOUR_MINUTE);
-- Returns: '2026-03-31 11:30:00'

-- Add 1 year and 6 months
SELECT DATE_ADD('2026-03-31', INTERVAL '1-6' YEAR_MONTH);
-- Returns: '2027-09-30'

-- Subtract 2 days and 12 hours
SELECT DATE_SUB('2026-03-31 12:00:00', INTERVAL '2 12' DAY_HOUR);
-- Returns: '2026-03-29 00:00:00'
```

---

## How DATE_ADD() and DATE_SUB() Work

```mermaid
flowchart LR
    A[Input date/datetime] --> B[Parse INTERVAL unit and value]
    B --> C{ADD or SUB?}
    C -- ADD --> D[Advance date by interval]
    C -- SUB --> E[Rewind date by interval]
    D --> F[Return adjusted DATETIME]
    E --> F
```

---

## Date Range Queries

```sql
-- Find orders placed in the last 30 days
SELECT id, order_date, total
FROM orders
WHERE order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY);

-- Find appointments in the next 7 days
SELECT id, patient_name, appointment_date
FROM appointments
WHERE appointment_date BETWEEN NOW() AND DATE_ADD(NOW(), INTERVAL 7 DAY);
```

---

## Expiry Date Calculations

```sql
CREATE TABLE subscriptions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    start_date DATE,
    duration_months INT
);

INSERT INTO subscriptions (user_id, start_date, duration_months) VALUES
(1, '2026-01-15', 12),
(2, '2026-03-01', 3),
(3, '2026-02-28', 6);

-- Calculate expiry dates
SELECT
    user_id,
    start_date,
    DATE_ADD(start_date, INTERVAL duration_months MONTH) AS expiry_date
FROM subscriptions;
```

---

## Using DATE_ADD() in UPDATE Statements

```sql
-- Extend all active subscription expiry dates by 1 month as a promotion
UPDATE subscriptions
SET start_date = DATE_ADD(start_date, INTERVAL 1 MONTH)
WHERE user_id IN (1, 2);
```

---

## Month-End Clamping

When adding months, MySQL clamps to the last valid day of the resulting month:

```sql
SELECT DATE_ADD('2026-01-31', INTERVAL 1 MONTH);
-- Returns: '2026-02-28' (February has no 31st)

SELECT DATE_ADD('2026-01-31', INTERVAL 2 MONTH);
-- Returns: '2026-03-31'
```

---

## ADDDATE() and SUBDATE() Aliases

`ADDDATE()` and `SUBDATE()` are aliases for `DATE_ADD()` and `DATE_SUB()`:

```sql
SELECT ADDDATE('2026-03-31', INTERVAL 10 DAY);
-- Same as DATE_ADD('2026-03-31', INTERVAL 10 DAY)

SELECT SUBDATE('2026-03-31', INTERVAL 10 DAY);
-- Same as DATE_SUB('2026-03-31', INTERVAL 10 DAY)

-- ADDDATE also accepts a plain integer as shorthand for days
SELECT ADDDATE('2026-03-31', 10);
-- Returns: '2026-04-10'
```

---

## NULL Handling

```sql
SELECT DATE_ADD(NULL, INTERVAL 7 DAY);
-- Returns: NULL

SELECT DATE_ADD('2026-03-31', INTERVAL NULL DAY);
-- Returns: NULL
```

---

## Summary

`DATE_ADD()` and `DATE_SUB()` are essential MySQL date arithmetic functions for calculating future and past dates. They support a wide range of interval units from microseconds to years, including compound units like `HOUR_MINUTE` and `YEAR_MONTH`. Use them in `WHERE` clauses for date range filtering, in `SELECT` lists for computed expiry and scheduling columns, and in `UPDATE` statements for bulk date adjustments. Be aware of month-end clamping behavior when working with months or quarters.
