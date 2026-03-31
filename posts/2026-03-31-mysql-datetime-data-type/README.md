# How to Use DATETIME Data Type in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Database, Data Type, Date, Storage

Description: Learn how the DATETIME type works in MySQL, its storage format, supported range, fractional seconds, and when to choose it over TIMESTAMP.

---

## What Is the DATETIME Data Type

`DATETIME` stores a combined date and time value in the format `YYYY-MM-DD HH:MM:SS`. It does not store timezone information - the value you insert is the value you get back, regardless of the server's timezone setting.

Storage: 5 bytes (plus 0-3 extra bytes for fractional seconds).
Range: `1000-01-01 00:00:00` to `9999-12-31 23:59:59`.

## Declaring a DATETIME Column

```sql
CREATE TABLE events (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title       VARCHAR(200) NOT NULL,
    start_at    DATETIME NOT NULL,
    end_at      DATETIME,
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

## Fractional Seconds

`DATETIME(n)` supports fractional seconds with precision from 0 to 6:

```sql
CREATE TABLE audit_log (
    id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    action     VARCHAR(100),
    logged_at  DATETIME(6) DEFAULT CURRENT_TIMESTAMP(6)
);

INSERT INTO audit_log (action) VALUES ('user_login');
SELECT id, action, logged_at FROM audit_log;
-- logged_at: 2026-03-31 14:22:05.123456
```

## Inserting DATETIME Values

```sql
INSERT INTO events (title, start_at, end_at)
VALUES ('Team Meeting', '2026-04-15 10:00:00', '2026-04-15 11:30:00');

-- Using NOW()
INSERT INTO events (title, start_at) VALUES ('Quick Standup', NOW());
```

## Querying DATETIME

```sql
-- Events in April 2026
SELECT * FROM events
WHERE start_at BETWEEN '2026-04-01 00:00:00' AND '2026-04-30 23:59:59';

-- Events starting today
SELECT * FROM events
WHERE DATE(start_at) = CURDATE();

-- Events in the next 7 days
SELECT * FROM events
WHERE start_at BETWEEN NOW() AND DATE_ADD(NOW(), INTERVAL 7 DAY);
```

## Extracting Date and Time Parts

```sql
SELECT
    title,
    DATE(start_at)           AS event_date,
    TIME(start_at)           AS event_time,
    YEAR(start_at)           AS yr,
    MONTH(start_at)          AS mo,
    DAY(start_at)            AS dy,
    HOUR(start_at)           AS hr,
    MINUTE(start_at)         AS mi,
    DAYNAME(start_at)        AS weekday
FROM events;
```

## Arithmetic with DATETIME

```sql
-- Duration of each event in minutes
SELECT
    title,
    TIMESTAMPDIFF(MINUTE, start_at, end_at) AS duration_min
FROM events
WHERE end_at IS NOT NULL;

-- Add 1 hour to a meeting
UPDATE events
SET end_at = DATE_ADD(end_at, INTERVAL 1 HOUR)
WHERE id = 1;
```

## Indexing DATETIME

```sql
ALTER TABLE events ADD INDEX idx_start_at (start_at);

-- Range queries benefit from the index
SELECT * FROM events
WHERE start_at >= '2026-01-01 00:00:00'
ORDER BY start_at;
```

## DATETIME vs TIMESTAMP

| Feature | DATETIME | TIMESTAMP |
|---|---|---|
| Range | 1000-9999 | 1970-2038 |
| Timezone | Not stored | Stored as UTC |
| Auto-update | Optional | Optional |
| Storage | 5 bytes | 4 bytes |

Choose `DATETIME` for future dates beyond 2038, when you manage timezone conversion in the application, or when you need consistent values regardless of the server timezone.

## Summary

`DATETIME` stores date and time without timezone conversion, making it ideal for events, schedules, and audit timestamps where you control timezone handling in the application. Use `DATETIME(6)` when microsecond precision is needed. For event-sourced systems where UTC storage and automatic timezone conversion matter, `TIMESTAMP` is a compact alternative.
