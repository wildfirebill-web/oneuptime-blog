# How to Use TIME_FORMAT() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Date Function, Time Function, Database

Description: Learn how to use MySQL TIME_FORMAT() to format time values into readable strings using format specifiers for hours, minutes, seconds, and AM/PM.

---

## What TIME_FORMAT() Does

`TIME_FORMAT()` converts a `TIME` or `DATETIME` value into a formatted string. It works like `DATE_FORMAT()` but is restricted to time-related format specifiers. It is useful when you need to display times in a specific layout for reports, UI output, or exports.

```sql
SELECT TIME_FORMAT('14:35:07', '%h:%i %p');
-- Result: 02:35 PM
```

The function signature is:

```sql
TIME_FORMAT(time, format)
```

- `time` - a `TIME`, `DATETIME`, or `TIMESTAMP` expression
- `format` - a format string using `%` specifiers

## Common Format Specifiers

| Specifier | Description | Example output |
|-----------|-------------|----------------|
| `%H` | Hour, 00-23 | `14` |
| `%h` | Hour, 01-12 | `02` |
| `%i` | Minutes, 00-59 | `35` |
| `%s` | Seconds, 00-59 | `07` |
| `%p` | AM or PM | `PM` |
| `%r` | Full 12-hour time (hh:mm:ss AM/PM) | `02:35:07 PM` |
| `%T` | Full 24-hour time (HH:mm:ss) | `14:35:07` |
| `%f` | Microseconds | `000000` |

## Basic Usage Examples

Format a literal time value:

```sql
SELECT TIME_FORMAT('09:05:03', '%H:%i:%s') AS formatted;
-- 09:05:03

SELECT TIME_FORMAT('09:05:03', '%h:%i %p') AS formatted;
-- 09:05 AM
```

Extract and format the time portion of a `DATETIME` column:

```sql
SELECT
  order_id,
  TIME_FORMAT(created_at, '%H:%i') AS order_time
FROM orders
WHERE DATE(created_at) = CURDATE();
```

## Using TIME_FORMAT with a Table

Suppose you have an `appointments` table with a `start_time TIME` column:

```sql
CREATE TABLE appointments (
  id INT AUTO_INCREMENT PRIMARY KEY,
  patient_name VARCHAR(100),
  start_time TIME NOT NULL
);

INSERT INTO appointments (patient_name, start_time) VALUES
  ('Alice', '08:30:00'),
  ('Bob', '13:15:00'),
  ('Carol', '17:45:00');
```

Display times in 12-hour format:

```sql
SELECT
  patient_name,
  TIME_FORMAT(start_time, '%h:%i %p') AS appointment_time
FROM appointments
ORDER BY start_time;
```

```text
Alice   08:30 AM
Bob     01:15 PM
Carol   05:45 PM
```

## Grouping by Hour

You can use `TIME_FORMAT()` together with `GROUP BY` to bucket records by hour:

```sql
SELECT
  TIME_FORMAT(created_at, '%H:00') AS hour_bucket,
  COUNT(*) AS total_orders
FROM orders
WHERE DATE(created_at) = '2026-03-31'
GROUP BY hour_bucket
ORDER BY hour_bucket;
```

This gives you an hourly breakdown without parsing strings manually.

## Handling NULL and Invalid Values

If the `time` argument is `NULL`, `TIME_FORMAT()` returns `NULL`:

```sql
SELECT TIME_FORMAT(NULL, '%H:%i');
-- NULL
```

If you pass a `DATE` value (with no time component), the function returns `00:00:00` formatted:

```sql
SELECT TIME_FORMAT('2026-03-31', '%H:%i:%s');
-- 00:00:00
```

Use `COALESCE()` to provide a fallback:

```sql
SELECT COALESCE(TIME_FORMAT(end_time, '%H:%i'), 'N/A') AS display_time
FROM sessions;
```

## Difference from DATE_FORMAT()

`DATE_FORMAT()` accepts date specifiers like `%Y`, `%m`, `%d` in addition to time specifiers. `TIME_FORMAT()` silently ignores date specifiers - they produce empty strings or zeroes. Always use `DATE_FORMAT()` when you need to format a complete datetime, and reserve `TIME_FORMAT()` for values that are purely time-based.

```sql
-- Correct for full datetime
SELECT DATE_FORMAT(NOW(), '%Y-%m-%d %H:%i:%s');

-- Correct for time only
SELECT TIME_FORMAT(CURTIME(), '%h:%i %p');
```

## Summary

`TIME_FORMAT()` is the dedicated MySQL function for converting time values into human-readable strings. Use `%H`/`%h` for hours, `%i` for minutes, `%s` for seconds, and `%p` for AM/PM. It integrates naturally with `GROUP BY` for time-based aggregations. For full datetime formatting, prefer `DATE_FORMAT()` instead.
