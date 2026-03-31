# How to Fix ERROR 1292 Incorrect Datetime Value in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, DateTime, Error, SQL Mode, Date

Description: Fix MySQL ERROR 1292 Incorrect datetime value by correcting date formats, adjusting sql_mode, and handling zero dates and timezone gaps properly.

---

MySQL ERROR 1292 occurs when a value being inserted or compared does not match the expected datetime format or falls outside the valid range. The error reads: `ERROR 1292 (22007): Incorrect datetime value: '0000-00-00' for column 'created_at'`.

## Common Triggers

- Inserting `0000-00-00` or `0000-00-00 00:00:00` when `NO_ZERO_DATE` is active in `sql_mode`
- Using the wrong date format like `31/12/2024` instead of `2024-12-31`
- Values in the DST (daylight saving time) gap that do not exist on the clock
- Inserting a date outside the valid range for the column type

## Check the Current sql_mode

```sql
SELECT @@sql_mode;
-- Or
SHOW VARIABLES LIKE 'sql_mode';
```

If `NO_ZERO_DATE` or `STRICT_TRANS_TABLES` is present, zero dates and out-of-range values will be rejected.

## Fix: Use the Correct Date Format

MySQL expects dates in `YYYY-MM-DD` format:

```sql
-- Incorrect
INSERT INTO events (event_date) VALUES ('31/12/2024');

-- Correct
INSERT INTO events (event_date) VALUES ('2024-12-31');

-- Use STR_TO_DATE to convert non-standard formats
INSERT INTO events (event_date)
VALUES (STR_TO_DATE('31/12/2024', '%d/%m/%Y'));
```

## Fix: Handle Zero Dates

If your application sends zero dates from legacy code, either fix the data or allow zero dates in `sql_mode`:

```sql
-- Remove NO_ZERO_DATE from sql_mode (session level)
SET SESSION sql_mode = REPLACE(@@sql_mode, 'NO_ZERO_DATE,', '');

-- Or allow NULL instead of zero dates
ALTER TABLE events MODIFY event_date DATETIME NULL;

-- Insert NULL instead of '0000-00-00'
INSERT INTO events (event_date) VALUES (NULL);
```

## Fix: Standardize sql_mode in my.cnf

For a consistent application experience, set `sql_mode` in the server configuration:

```text
[mysqld]
sql_mode = ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
```

Removing `NO_ZERO_DATE` from the list allows zero dates without errors. Restart MySQL after changes.

## Fix: DST Gap Values

In timezones with daylight saving transitions, certain timestamps literally do not exist. For example, `2024-03-10 02:30:00` does not exist in US/Eastern. Use UTC for storage to avoid this:

```sql
-- Store timestamps in UTC
SET time_zone = '+00:00';
INSERT INTO logs (created_at) VALUES (UTC_TIMESTAMP());

-- Convert on retrieval
SELECT CONVERT_TZ(created_at, '+00:00', 'America/New_York') AS local_time
FROM logs;
```

## Fix: Bulk Data Correction

If you have existing zero-date rows that need fixing, update them:

```sql
-- Find rows with zero dates
SELECT id, event_date FROM events WHERE event_date = '0000-00-00';

-- Replace with NULL or a valid default
UPDATE events
SET event_date = NULL
WHERE event_date = '0000-00-00';
```

## Summary

ERROR 1292 is caused by invalid datetime values reaching MySQL. Fix it by using `YYYY-MM-DD` date format, replacing zero dates with NULL or valid dates, and choosing an appropriate `sql_mode` that matches your application's data. Storing all timestamps in UTC eliminates DST-related invalid datetime issues entirely.
