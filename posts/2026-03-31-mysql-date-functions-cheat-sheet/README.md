# MySQL Date Functions Cheat Sheet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Date Function, Query, Cheat Sheet

Description: Quick reference for MySQL date and time functions including NOW, DATE_ADD, DATEDIFF, DATE_FORMAT, EXTRACT, STR_TO_DATE, and TIMESTAMPDIFF with examples.

---

## Current Date and Time

```sql
SELECT NOW();              -- '2026-03-31 14:22:05' (datetime)
SELECT CURDATE();          -- '2026-03-31' (date only)
SELECT CURTIME();          -- '14:22:05' (time only)
SELECT UTC_TIMESTAMP();    -- UTC datetime
SELECT UNIX_TIMESTAMP();   -- Unix epoch seconds
```

## Arithmetic: Add and Subtract

```sql
-- DATE_ADD / ADDDATE
SELECT DATE_ADD('2026-01-01', INTERVAL 30 DAY);    -- '2026-01-31'
SELECT DATE_ADD(NOW(), INTERVAL 1 MONTH);
SELECT DATE_ADD(NOW(), INTERVAL 2 HOUR);

-- DATE_SUB / SUBDATE
SELECT DATE_SUB(CURDATE(), INTERVAL 7 DAY);
SELECT SUBDATE(CURDATE(), 7);
```

## Intervals Reference

```text
MICROSECOND, SECOND, MINUTE, HOUR
DAY, WEEK, MONTH, QUARTER, YEAR
SECOND_MICROSECOND, MINUTE_SECOND, HOUR_MINUTE
DAY_HOUR, YEAR_MONTH
```

## Difference Between Dates

```sql
SELECT DATEDIFF('2026-12-31', '2026-01-01');          -- 364 (days)
SELECT TIMEDIFF('23:00:00', '10:00:00');              -- '13:00:00'

-- TIMESTAMPDIFF (flexible unit)
SELECT TIMESTAMPDIFF(MONTH, '2025-01-01', '2026-03-31');  -- 14
SELECT TIMESTAMPDIFF(DAY,   '2026-01-01', '2026-03-31');  -- 89
SELECT TIMESTAMPDIFF(HOUR,  '2026-03-31 08:00', NOW());
```

## Extracting Parts

```sql
SELECT YEAR('2026-03-31');       -- 2026
SELECT MONTH('2026-03-31');      -- 3
SELECT DAY('2026-03-31');        -- 31
SELECT HOUR(NOW());
SELECT MINUTE(NOW());
SELECT SECOND(NOW());
SELECT DAYOFWEEK('2026-03-31');  -- 3 (1=Sunday)
SELECT DAYOFYEAR('2026-03-31');  -- 90
SELECT WEEK('2026-03-31');       -- week number
SELECT QUARTER('2026-03-31');    -- 1

-- EXTRACT (SQL standard style)
SELECT EXTRACT(YEAR  FROM '2026-03-31');
SELECT EXTRACT(MONTH FROM '2026-03-31');
```

## Formatting Dates

```sql
SELECT DATE_FORMAT(NOW(), '%Y-%m-%d');           -- '2026-03-31'
SELECT DATE_FORMAT(NOW(), '%d/%m/%Y %H:%i:%s'); -- '31/03/2026 14:22:05'
SELECT DATE_FORMAT(NOW(), '%W, %M %e, %Y');      -- 'Tuesday, March 31, 2026'
```

Common format specifiers:
```text
%Y - 4-digit year     %y - 2-digit year
%m - month (01-12)    %M - month name
%d - day (01-31)      %e - day (1-31)
%H - hour 24h         %h - hour 12h
%i - minutes          %s - seconds
%W - weekday name     %p - AM/PM
```

## Parsing Strings into Dates

```sql
SELECT STR_TO_DATE('31/03/2026', '%d/%m/%Y');   -- '2026-03-31'
SELECT STR_TO_DATE('March 31 2026', '%M %d %Y');
```

## Conversion Utilities

```sql
SELECT FROM_UNIXTIME(1743407325);            -- datetime from epoch
SELECT UNIX_TIMESTAMP('2026-03-31 12:00:00'); -- epoch from datetime

SELECT DATE(NOW());          -- strip time portion
SELECT TIME(NOW());          -- strip date portion
SELECT LAST_DAY('2026-02-01');  -- '2026-02-28'
```

## Practical Query Examples

```sql
-- Orders from the last 30 days
SELECT * FROM orders
WHERE created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY);

-- Group by month
SELECT DATE_FORMAT(created_at, '%Y-%m') AS month,
       COUNT(*) AS orders
FROM orders
GROUP BY month
ORDER BY month;
```

## Summary

MySQL date functions cover creation (NOW, CURDATE), arithmetic (DATE_ADD, DATE_SUB, DATEDIFF, TIMESTAMPDIFF), extraction (YEAR, MONTH, EXTRACT), formatting (DATE_FORMAT), and parsing (STR_TO_DATE). Mastering these eliminates most date manipulation in application code and enables efficient index-friendly filtering on datetime columns.
