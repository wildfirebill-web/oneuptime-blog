# How to Use UTC_DATE() and UTC_TIME() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Utc Date, Utc Time, Timezone, Date Functions

Description: Learn how to use MySQL's UTC_DATE(), UTC_TIME(), and UTC_TIMESTAMP() functions to work with Coordinated Universal Time in your queries.

---

## Overview

MySQL provides three dedicated functions for retrieving the current date and time in Coordinated Universal Time (UTC): `UTC_DATE()`, `UTC_TIME()`, and `UTC_TIMESTAMP()`. These complement `NOW()` and `CURDATE()`, which return values in the server's local time zone. Using UTC functions ensures consistency in distributed systems regardless of server location.

## Function Reference

```sql
UTC_DATE()        -- returns current UTC date as DATE (YYYY-MM-DD)
UTC_TIME()        -- returns current UTC time as TIME (HH:MM:SS)
UTC_TIMESTAMP()   -- returns current UTC datetime as DATETIME (YYYY-MM-DD HH:MM:SS)
```

## Basic Examples

```sql
-- Get current UTC date
SELECT UTC_DATE();            -- e.g., 2024-06-15

-- Get current UTC time
SELECT UTC_TIME();            -- e.g., 12:30:45

-- Get current UTC datetime
SELECT UTC_TIMESTAMP();       -- e.g., 2024-06-15 12:30:45

-- Compare with local server time
SELECT NOW(), UTC_TIMESTAMP(), TIMEDIFF(NOW(), UTC_TIMESTAMP()) AS utc_offset;
```

## Storing Records with UTC Timestamps

```sql
CREATE TABLE audit_log (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  action VARCHAR(100),
  performed_by VARCHAR(50),
  created_utc DATETIME DEFAULT (UTC_TIMESTAMP())
);

-- Insert using UTC_TIMESTAMP()
INSERT INTO audit_log (action, performed_by)
VALUES ('user.login', 'alice@example.com');

-- Verify
SELECT * FROM audit_log;
```

## Filtering by UTC Date

```sql
-- Records created today in UTC
SELECT * FROM events
WHERE DATE(created_utc) = UTC_DATE();

-- Records from the last 24 hours in UTC
SELECT * FROM events
WHERE created_utc >= UTC_TIMESTAMP() - INTERVAL 24 HOUR;

-- Records from a specific UTC date
SELECT * FROM orders
WHERE DATE(order_utc) = '2024-06-15';
```

## UTC_DATE() in Date Arithmetic

```sql
-- Days since a fixed UTC date
SELECT DATEDIFF(UTC_DATE(), '2024-01-01') AS days_since_new_year;

-- Next UTC month start
SELECT DATE_FORMAT(UTC_DATE() + INTERVAL 1 MONTH, '%Y-%m-01') AS next_month_start;
```

## Comparing UTC_TIME() Ranges

```sql
-- Find events that occur during business hours UTC (09:00 - 17:00)
SELECT * FROM scheduled_events
WHERE UTC_TIME() BETWEEN '09:00:00' AND '17:00:00';
```

## Fractional Seconds Precision

```sql
-- UTC functions support fractional seconds in MySQL 5.6+
SELECT UTC_TIMESTAMP(6);   -- e.g., 2024-06-15 12:30:45.123456
SELECT UTC_TIME(3);        -- e.g., 12:30:45.123
```

## UTC vs Local Time

```sql
-- If the server is set to a non-UTC timezone, NOW() and UTC_TIMESTAMP() differ
SET time_zone = 'America/New_York';
SELECT NOW() AS local_time, UTC_TIMESTAMP() AS utc_time;
-- local_time: 2024-06-15 08:30:00, utc_time: 2024-06-15 12:30:00
```

## Best Practice: Always Store UTC

```sql
-- Create a table always storing UTC
CREATE TABLE global_events (
  id INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(200),
  starts_at DATETIME COMMENT 'always UTC',
  created_at DATETIME DEFAULT (UTC_TIMESTAMP())
);

INSERT INTO global_events (title, starts_at)
VALUES ('Global Webinar', UTC_TIMESTAMP() + INTERVAL 7 DAY);
```

## Summary

`UTC_DATE()`, `UTC_TIME()`, and `UTC_TIMESTAMP()` return the current date, time, and datetime in UTC regardless of the MySQL server's time zone setting. Always store datetimes as UTC in globally distributed applications, and use these functions to generate UTC values consistently. Display local time using `CONVERT_TZ()` at query time.
