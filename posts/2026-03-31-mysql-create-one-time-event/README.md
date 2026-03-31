# How to Create a One-Time Event in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Event Scheduler, One-Time Event, Automation, Scheduled Job

Description: Learn how to create a one-time MySQL event that executes a SQL statement at a specific date and time, and how to manage it after it runs.

---

A one-time MySQL event runs exactly once at a specified point in the future and then either drops itself or stays in the event list as disabled, depending on how you declare it. One-time events are useful for deferred tasks like sending a batch notification, running a data migration, or purging records at a predetermined time.

## Basic Syntax

```sql
CREATE EVENT event_name
ON SCHEDULE AT timestamp
DO
    sql_statement;
```

The scheduler must be enabled before the event can run:

```sql
SHOW VARIABLES LIKE 'event_scheduler';
SET GLOBAL event_scheduler = ON;
```

## Example 1 - Run at a Specific Date and Time

Schedule a cleanup job to run once at midnight on April 1, 2026:

```sql
CREATE EVENT evt_cleanup_old_sessions
ON SCHEDULE AT '2026-04-01 00:00:00'
DO
    DELETE FROM sessions WHERE last_active < NOW() - INTERVAL 30 DAY;
```

## Example 2 - Run Once After a Delay

Use `NOW() + INTERVAL` to schedule relative to the current time:

```sql
CREATE EVENT evt_send_reminder_emails
ON SCHEDULE AT NOW() + INTERVAL 2 HOUR
DO
    UPDATE notification_queue SET status = 'pending' WHERE type = 'reminder';
```

## Example 3 - Multi-Statement Event Body

For more than one statement, wrap the body in a `BEGIN ... END` block and change the delimiter:

```sql
DELIMITER //

CREATE EVENT evt_archive_completed_orders
ON SCHEDULE AT '2026-04-15 02:00:00'
DO
BEGIN
    INSERT INTO orders_archive
    SELECT * FROM orders WHERE status = 'completed' AND created_at < '2026-01-01';

    DELETE FROM orders
    WHERE status = 'completed' AND created_at < '2026-01-01';
END //

DELIMITER ;
```

## What Happens After the Event Runs

By default (`ON COMPLETION NOT PRESERVE`), MySQL drops the event from the event table after it executes. To keep the event definition for reference or logging:

```sql
CREATE EVENT evt_one_time_task
ON SCHEDULE AT NOW() + INTERVAL 1 HOUR
ON COMPLETION PRESERVE
DO
    INSERT INTO task_log (task, ran_at) VALUES ('one_time_task', NOW());
```

The event becomes disabled (not dropped) after it runs, so you can inspect it later.

## Viewing the Event

```sql
SHOW EVENTS LIKE 'evt_cleanup_old_sessions';

SELECT EVENT_NAME, STATUS, EXECUTE_AT, LAST_EXECUTED
FROM information_schema.EVENTS
WHERE EVENT_SCHEMA = 'mydb';
```

## Dropping a One-Time Event

If you need to cancel before it runs:

```sql
DROP EVENT IF EXISTS evt_cleanup_old_sessions;
```

## Summary

Create a one-time MySQL event with `ON SCHEDULE AT timestamp` to run a SQL statement at a future point. Use `NOW() + INTERVAL` for relative scheduling, add `ON COMPLETION PRESERVE` to retain the event definition after it runs, and drop the event if you need to cancel it before execution.
