# How to Create a Recurring Event in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Event Scheduler, Recurring Event, EVERY, Automation

Description: Learn how to create recurring MySQL events that run on a schedule using EVERY with interval units, with start/end times and practical examples.

---

A recurring MySQL event uses `ON SCHEDULE EVERY interval` to run repeatedly at a fixed frequency. It is the MySQL equivalent of a cron job - useful for periodic cleanups, summary aggregations, heartbeat inserts, and other background maintenance tasks.

## Basic Syntax

```sql
CREATE EVENT event_name
ON SCHEDULE EVERY interval_value interval_unit
[STARTS start_timestamp]
[ENDS end_timestamp]
DO
    sql_statement;
```

Common interval units: `SECOND`, `MINUTE`, `HOUR`, `DAY`, `WEEK`, `MONTH`, `QUARTER`, `YEAR`.

## Example 1 - Run Every Hour

Delete expired sessions every hour:

```sql
CREATE EVENT evt_purge_expired_sessions
ON SCHEDULE EVERY 1 HOUR
DO
    DELETE FROM sessions WHERE expires_at < NOW();
```

## Example 2 - Run Every Day at Midnight

Use `STARTS` to fix the first execution time:

```sql
CREATE EVENT evt_daily_summary
ON SCHEDULE EVERY 1 DAY
STARTS '2026-04-01 00:00:00'
DO
    INSERT INTO daily_stats (stat_date, order_count, revenue)
    SELECT CURDATE() - INTERVAL 1 DAY,
           COUNT(*),
           SUM(total)
    FROM orders
    WHERE DATE(created_at) = CURDATE() - INTERVAL 1 DAY;
```

## Example 3 - Run Every 5 Minutes with an End Date

```sql
DELIMITER //

CREATE EVENT evt_sync_cache
ON SCHEDULE EVERY 5 MINUTE
STARTS NOW()
ENDS '2026-12-31 23:59:59'
DO
BEGIN
    UPDATE product_cache pc
    INNER JOIN products p ON pc.product_id = p.product_id
    SET pc.cached_price = p.price,
        pc.updated_at   = NOW()
    WHERE p.updated_at > pc.updated_at;
END //

DELIMITER ;
```

The event automatically stops and drops itself (or becomes disabled, depending on `ON COMPLETION`) after the `ENDS` timestamp passes.

## Using ON COMPLETION PRESERVE

By default a recurring event that reaches its `ENDS` date is dropped. To keep the definition as disabled:

```sql
CREATE EVENT evt_monthly_archive
ON SCHEDULE EVERY 1 MONTH
STARTS '2026-05-01 02:00:00'
ENDS   '2027-05-01 02:00:00'
ON COMPLETION PRESERVE
DO
    CALL sp_archive_old_records();
```

## Viewing Recurring Events

```sql
SHOW EVENTS;

SELECT EVENT_NAME, STATUS, INTERVAL_VALUE, INTERVAL_FIELD, STARTS, ENDS, LAST_EXECUTED
FROM information_schema.EVENTS
WHERE EVENT_SCHEMA = 'mydb';
```

## Altering and Dropping

Change the interval without dropping and recreating:

```sql
ALTER EVENT evt_purge_expired_sessions
ON SCHEDULE EVERY 30 MINUTE;
```

Drop when no longer needed:

```sql
DROP EVENT IF EXISTS evt_purge_expired_sessions;
```

## Summary

Create a recurring MySQL event with `ON SCHEDULE EVERY n unit`, optionally combined with `STARTS` and `ENDS` to define the active window. Use `ON COMPLETION PRESERVE` to retain the event definition after it expires, and monitor execution times and status through `information_schema.EVENTS`.
