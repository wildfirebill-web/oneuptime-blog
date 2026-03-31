# How to Use TIMESTAMP Data Type in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Database, Data Type, Date, Storage

Description: Learn how the TIMESTAMP type works in MySQL, its UTC storage, auto-update behavior, 2038 limit, and when to choose it over DATETIME.

---

## What Is the TIMESTAMP Data Type

`TIMESTAMP` stores a point in time as the number of seconds elapsed since the Unix epoch (1970-01-01 00:00:00 UTC). MySQL converts inserted values from the current session timezone to UTC for storage, and converts them back to the session timezone on retrieval.

Storage: 4 bytes (plus 0-3 bytes for fractional seconds).
Range: `1970-01-01 00:00:01 UTC` to `2038-01-19 03:14:07 UTC`.

## Declaring a TIMESTAMP Column

```sql
CREATE TABLE orders (
    id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    status     CHAR(1) NOT NULL DEFAULT 'P',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

## Auto-Update Behavior

The `ON UPDATE CURRENT_TIMESTAMP` clause tells MySQL to automatically set the column to the current UTC time whenever the row is updated - a common pattern for audit trails:

```sql
INSERT INTO orders (status) VALUES ('P');
-- created_at and updated_at are set to NOW()

UPDATE orders SET status = 'S' WHERE id = 1;
-- updated_at is automatically refreshed
```

## Timezone Conversion

```sql
-- Check session timezone
SELECT @@session.time_zone;

-- Insert in a specific timezone context
SET time_zone = 'America/New_York';
INSERT INTO orders (status) VALUES ('P');
-- MySQL stores the UTC equivalent

-- Retrieve: MySQL converts UTC back to America/New_York
SELECT id, created_at FROM orders;
```

## Fractional Seconds

```sql
CREATE TABLE clicks (
    id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    element_id VARCHAR(100),
    clicked_at TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6)
);

SELECT id, element_id, clicked_at FROM clicks ORDER BY clicked_at DESC LIMIT 5;
-- clicked_at: 2026-03-31 14:22:05.123456
```

## Querying TIMESTAMP

```sql
-- Orders placed today (in the session timezone)
SELECT * FROM orders
WHERE DATE(created_at) = CURDATE();

-- Orders in a UTC range
SET time_zone = '+00:00';
SELECT * FROM orders
WHERE created_at BETWEEN '2026-03-01 00:00:00' AND '2026-03-31 23:59:59';
```

## Unix Timestamp Conversion

```sql
-- Convert TIMESTAMP to Unix epoch integer
SELECT id, UNIX_TIMESTAMP(created_at) AS epoch_seconds FROM orders;

-- Convert Unix integer back to TIMESTAMP
SELECT FROM_UNIXTIME(1743370005) AS ts;
```

## The Year 2038 Problem

`TIMESTAMP` cannot store dates after `2038-01-19 03:14:07 UTC`. Plan migrations early if your application will operate past that date:

```sql
-- Check if you have TIMESTAMP columns
SELECT TABLE_NAME, COLUMN_NAME
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'mydb' AND DATA_TYPE = 'timestamp';

-- Migration: change to DATETIME to extend range past 2038
ALTER TABLE orders
    MODIFY created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    MODIFY updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP;
```

## TIMESTAMP vs DATETIME

| Feature | TIMESTAMP | DATETIME |
|---|---|---|
| Timezone | Converts to/from UTC | Stored as-is |
| Range | 1970-2038 | 1000-9999 |
| Storage | 4 bytes | 5 bytes |
| Auto-update | Supported | Supported |

Use `TIMESTAMP` when you need automatic UTC storage and timezone-aware retrieval for audit fields like `created_at` and `updated_at`. Use `DATETIME` for dates beyond 2038 or when you handle timezone conversion in the application.

## Summary

`TIMESTAMP` is compact, timezone-aware, and supports automatic update on row changes, making it the standard choice for `created_at` and `updated_at` audit columns. Be aware of the 2038 upper limit and plan schema changes well in advance. For application dates that need to extend beyond 2038 or that should not be affected by server timezone changes, use `DATETIME`.
