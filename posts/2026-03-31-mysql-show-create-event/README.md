# How to Use SHOW CREATE EVENT in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Event, Scheduler, Administration

Description: Learn how to use SHOW CREATE EVENT in MySQL to retrieve the DDL statement for a scheduled event, useful for backups, migrations, and auditing.

---

## What Is SHOW CREATE EVENT?

MySQL's Event Scheduler allows you to run scheduled SQL tasks automatically. The `SHOW CREATE EVENT` statement retrieves the exact `CREATE EVENT` DDL for an existing event. This is useful when you need to document, migrate, or recreate events on another server.

## Basic Syntax

```sql
SHOW CREATE EVENT event_name;
SHOW CREATE EVENT db_name.event_name;
```

You must specify the event name. If you are not in the database where the event lives, qualify it with the schema name.

## Creating a Sample Event

First, make sure the event scheduler is running:

```sql
SET GLOBAL event_scheduler = ON;
SHOW VARIABLES LIKE 'event_scheduler';
```

Now create a simple recurring event:

```sql
CREATE EVENT cleanup_old_sessions
ON SCHEDULE EVERY 1 HOUR
STARTS CURRENT_TIMESTAMP
COMMENT 'Removes sessions older than 24 hours'
DO
  DELETE FROM sessions WHERE created_at < NOW() - INTERVAL 24 HOUR;
```

## Retrieving the Event DDL

```sql
SHOW CREATE EVENT cleanup_old_sessions\G
```

Sample output:

```text
*************************** 1. row ***************************
               Event: cleanup_old_sessions
            sql_mode: ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,...
           time_zone: SYSTEM
        Create Event: CREATE DEFINER=`root`@`localhost` EVENT `cleanup_old_sessions`
                      ON SCHEDULE EVERY 1 HOUR
                      STARTS '2026-03-31 00:00:00'
                      ON COMPLETION NOT PRESERVE
                      ENABLE
                      COMMENT 'Removes sessions older than 24 hours'
                      DO DELETE FROM sessions WHERE created_at < NOW() - INTERVAL 24 HOUR
character_set_client: utf8mb4
collation_connection: utf8mb4_0900_ai_ci
  Database Collation: utf8mb4_0900_ai_ci
```

## Listing All Events First

Before using `SHOW CREATE EVENT`, list available events to get the exact names:

```sql
-- Show events in the current database
SHOW EVENTS;

-- Show events in a specific database
SHOW EVENTS FROM mydb;

-- Query information_schema for more detail
SELECT EVENT_NAME, EVENT_SCHEMA, STATUS, INTERVAL_VALUE, INTERVAL_FIELD
FROM information_schema.EVENTS
WHERE EVENT_SCHEMA = 'mydb';
```

## Using SHOW CREATE EVENT for Migration

When moving events to another server:

```sql
-- On source: retrieve the DDL
SHOW CREATE EVENT mydb.cleanup_old_sessions\G

-- Copy the "Create Event" value and run it on the target server
-- Make sure the event scheduler is enabled on the target too
SET GLOBAL event_scheduler = ON;
```

## Modifying an Existing Event

After reviewing the DDL, you can modify it using `ALTER EVENT`:

```sql
-- Change the schedule to run every 30 minutes
ALTER EVENT cleanup_old_sessions
ON SCHEDULE EVERY 30 MINUTE;

-- Disable an event temporarily
ALTER EVENT cleanup_old_sessions DISABLE;

-- Re-enable it
ALTER EVENT cleanup_old_sessions ENABLE;
```

## Required Privileges

To use `SHOW CREATE EVENT`, you need `EVENT` privilege on the database:

```sql
GRANT EVENT ON mydb.* TO 'dba_user'@'localhost';
```

## Summary

`SHOW CREATE EVENT` is a straightforward way to extract the full DDL of a MySQL scheduled event. Use it when documenting your database, scripting migrations, or auditing what automated tasks are configured. Combine it with `SHOW EVENTS` and `information_schema.EVENTS` for a complete picture of your event scheduler setup.
