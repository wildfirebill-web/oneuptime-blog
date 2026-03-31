# How to Disable an Event Without Dropping It in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Event Scheduler, Disable Event, ALTER EVENT, Maintenance

Description: Learn how to temporarily disable a MySQL event using ALTER EVENT DISABLE so it stops running without losing its definition, and how to re-enable it later.

---

Sometimes you need to stop a scheduled event temporarily - during maintenance windows, deployments, or debugging - without permanently removing it. MySQL's `ALTER EVENT ... DISABLE` command pauses an event while preserving its definition for easy re-activation.

## Disabling an Event

```sql
ALTER EVENT evt_daily_cleanup DISABLE;
```

The event immediately stops being scheduled. Any currently running invocation continues until it completes, but no new executions are triggered.

Verify the status change:

```sql
SHOW EVENTS LIKE 'evt_daily_cleanup';
-- Status column shows: DISABLED
```

Or query `information_schema`:

```sql
SELECT EVENT_NAME, STATUS, LAST_EXECUTED
FROM information_schema.EVENTS
WHERE EVENT_SCHEMA = 'mydb'
  AND EVENT_NAME = 'evt_daily_cleanup';
```

## Re-enabling an Event

```sql
ALTER EVENT evt_daily_cleanup ENABLE;
```

The event resumes its schedule immediately. For a recurring event, the next execution is calculated from the current time relative to the schedule interval, not from when it was disabled.

## Disabling Multiple Events for a Maintenance Window

Generate `ALTER EVENT ... DISABLE` statements for all enabled events:

```sql
SELECT CONCAT('ALTER EVENT `', EVENT_SCHEMA, '`.`', EVENT_NAME, '` DISABLE;') AS stmt
FROM information_schema.EVENTS
WHERE EVENT_SCHEMA = 'mydb'
  AND STATUS = 'ENABLED';
```

Run the generated statements, perform your maintenance, then re-enable:

```sql
SELECT CONCAT('ALTER EVENT `', EVENT_SCHEMA, '`.`', EVENT_NAME, '` ENABLE;') AS stmt
FROM information_schema.EVENTS
WHERE EVENT_SCHEMA = 'mydb'
  AND STATUS = 'DISABLED';
```

## Using a Stored Procedure for Bulk Enable/Disable

```sql
DELIMITER //

CREATE PROCEDURE toggle_all_events(IN p_schema VARCHAR(64), IN p_action ENUM('ENABLE','DISABLE'))
BEGIN
    DECLARE v_event VARCHAR(64);
    DECLARE done INT DEFAULT 0;
    DECLARE cur CURSOR FOR
        SELECT EVENT_NAME FROM information_schema.EVENTS WHERE EVENT_SCHEMA = p_schema;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

    OPEN cur;
    lp: LOOP
        FETCH cur INTO v_event;
        IF done THEN LEAVE lp; END IF;
        SET @sql = CONCAT('ALTER EVENT `', p_schema, '`.`', v_event, '` ', p_action);
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;
    END LOOP;
    CLOSE cur;
END //

DELIMITER ;

-- Disable all events in mydb
CALL toggle_all_events('mydb', 'DISABLE');

-- Re-enable all events in mydb
CALL toggle_all_events('mydb', 'ENABLE');
```

## DISABLE vs DISABLE ON SLAVE

In a replication setup, `DISABLE ON SLAVE` marks an event as disabled only on replica servers, allowing it to run on the source while suppressing execution on replicas:

```sql
ALTER EVENT evt_daily_cleanup DISABLE ON SLAVE;
```

This is useful when you only want the event to run once across your cluster (on the source) and not be duplicated on every replica.

## Summary

Use `ALTER EVENT event_name DISABLE` to pause a MySQL scheduled event without removing it, and `ALTER EVENT event_name ENABLE` to resume it. This is safer than `DROP EVENT` for temporary pauses during maintenance, and `DISABLE ON SLAVE` provides replication-aware scheduling for HA environments.
