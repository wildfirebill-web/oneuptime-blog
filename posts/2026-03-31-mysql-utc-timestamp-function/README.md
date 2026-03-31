# How to Use UTC_TIMESTAMP() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Date Function, UTC, Database

Description: Learn how to use MySQL UTC_TIMESTAMP() to get the current UTC datetime for storing consistent timestamps independent of the server time zone setting.

---

## What Is UTC_TIMESTAMP()?

`UTC_TIMESTAMP()` returns the current date and time in Coordinated Universal Time (UTC) as a `DATETIME` value, regardless of the MySQL server's configured time zone.

**Syntax:**

```sql
UTC_TIMESTAMP()
UTC_TIMESTAMP(fsp)
```

The optional `fsp` argument specifies fractional seconds precision (0-6). When omitted, no fractional seconds are included.

## Basic Usage

```sql
SELECT UTC_TIMESTAMP();
-- Returns: 2026-03-31 12:45:00 (example UTC time)

SELECT UTC_TIMESTAMP(3);
-- Returns: 2026-03-31 12:45:00.123 (with milliseconds)

-- Compare with NOW() when server is in a different timezone
SET time_zone = 'America/New_York';

SELECT NOW() AS local_time, UTC_TIMESTAMP() AS utc_time;
-- local_time: 2026-03-31 08:45:00
-- utc_time:   2026-03-31 12:45:00
```

## Why Use UTC_TIMESTAMP()?

Applications serving global users should store timestamps in UTC to avoid:
- Ambiguous times during daylight saving transitions
- Incorrect comparisons between timestamps from different time zones
- Data corruption when the server time zone is changed

`UTC_TIMESTAMP()` guarantees the stored value is always UTC-based regardless of server or session time zone.

## Storing UTC Timestamps on Insert

```sql
CREATE TABLE events (
  event_id   INT AUTO_INCREMENT PRIMARY KEY,
  event_name VARCHAR(255),
  created_at DATETIME DEFAULT NULL
);

INSERT INTO events (event_name, created_at)
VALUES ('User signup', UTC_TIMESTAMP());
```

Compare this with using `NOW()` or `CURRENT_TIMESTAMP`, which store the session time zone's local time.

## Using UTC_TIMESTAMP() as a Column Default

```sql
CREATE TABLE audit_log (
  log_id    INT AUTO_INCREMENT PRIMARY KEY,
  action    VARCHAR(100),
  logged_at DATETIME NOT NULL DEFAULT (UTC_TIMESTAMP())
);
```

Note: In MySQL 8.0+, expression defaults require parentheses.

## Calculating Time Differences in UTC

Find events that occurred in the last 24 hours, using UTC:

```sql
SELECT event_id, event_name, created_at
FROM events
WHERE created_at >= UTC_TIMESTAMP() - INTERVAL 24 HOUR;
```

Find how many seconds ago an event occurred:

```sql
SELECT
  event_id,
  event_name,
  TIMESTAMPDIFF(SECOND, created_at, UTC_TIMESTAMP()) AS seconds_ago
FROM events
ORDER BY created_at DESC
LIMIT 10;
```

## Difference Between UTC Functions

MySQL offers three UTC date/time functions:

| Function         | Returns            | Type     | Example                     |
|------------------|--------------------|----------|-----------------------------|
| `UTC_DATE()`     | Current UTC date   | DATE     | 2026-03-31                  |
| `UTC_TIME()`     | Current UTC time   | TIME     | 12:45:00                    |
| `UTC_TIMESTAMP()`| Current UTC datetime | DATETIME | 2026-03-31 12:45:00        |

```sql
SELECT
  UTC_DATE()      AS utc_date,
  UTC_TIME()      AS utc_time,
  UTC_TIMESTAMP() AS utc_datetime;
```

## Converting UTC_TIMESTAMP() to Local Time

Use `CONVERT_TZ()` to display the UTC timestamp in a local time zone:

```sql
SELECT
  event_name,
  created_at AS utc_time,
  CONVERT_TZ(created_at, 'UTC', 'America/Los_Angeles') AS pacific_time,
  CONVERT_TZ(created_at, 'UTC', 'Europe/London') AS london_time
FROM events;
```

## Using UTC_TIMESTAMP() in UPDATE Statements

Stamp records with the UTC time of last modification:

```sql
UPDATE orders
SET
  status = 'shipped',
  shipped_at = UTC_TIMESTAMP()
WHERE order_id = 1001;
```

## Stored Procedure Example

```sql
DELIMITER //
CREATE PROCEDURE log_action(IN p_action VARCHAR(100))
BEGIN
  INSERT INTO audit_log (action, logged_at)
  VALUES (p_action, UTC_TIMESTAMP());
END //
DELIMITER ;

CALL log_action('password_changed');
```

## Summary

`UTC_TIMESTAMP()` returns the current UTC datetime in MySQL, making it the preferred function for recording timestamps in globally-distributed applications. It ensures all stored datetimes are time zone-neutral, preventing bugs caused by daylight saving changes or varying server configurations. Use `CONVERT_TZ()` to translate stored UTC timestamps to any local time zone when displaying them to users.
