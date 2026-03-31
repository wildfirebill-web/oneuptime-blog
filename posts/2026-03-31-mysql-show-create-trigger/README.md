# How to Use SHOW CREATE TRIGGER in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Trigger, DDL, Administration

Description: Learn how to use SHOW CREATE TRIGGER in MySQL to retrieve the full trigger definition for documentation, migration, and debugging purposes.

---

## What Is SHOW CREATE TRIGGER?

Triggers in MySQL are stored programs that automatically execute in response to `INSERT`, `UPDATE`, or `DELETE` events on a table. The `SHOW CREATE TRIGGER` statement retrieves the exact `CREATE TRIGGER` DDL for any existing trigger. This is invaluable for auditing, schema migration, and understanding the side effects of DML operations.

## Basic Syntax

```sql
SHOW CREATE TRIGGER trigger_name;
```

The trigger name must exist in the current database or you should switch to the correct database first.

## Creating a Sample Trigger

```sql
DELIMITER $$

CREATE TRIGGER log_user_update
AFTER UPDATE ON users
FOR EACH ROW
BEGIN
  INSERT INTO audit_log (table_name, record_id, changed_at, changed_by)
  VALUES ('users', NEW.id, NOW(), USER());
END$$

DELIMITER ;
```

## Retrieving the Trigger DDL

```sql
SHOW CREATE TRIGGER log_user_update\G
```

Sample output:

```text
*************************** 1. row ***************************
               Trigger: log_user_update
              sql_mode: ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,...
SQL Original Statement: CREATE DEFINER=`root`@`localhost` TRIGGER `log_user_update`
                        AFTER UPDATE ON `users`
                        FOR EACH ROW
                        BEGIN
                          INSERT INTO audit_log (table_name, record_id, changed_at, changed_by)
                          VALUES ('users', NEW.id, NOW(), USER());
                        END
  character_set_client: utf8mb4
  collation_connection: utf8mb4_0900_ai_ci
    Database Collation: utf8mb4_0900_ai_ci
            Created: 2026-03-31 10:00:00.00
```

## Listing All Triggers on a Table

```sql
-- Show all triggers in the current database
SHOW TRIGGERS;

-- Filter by table name
SHOW TRIGGERS WHERE `Table` = 'users';

-- Query information_schema for complete detail
SELECT TRIGGER_NAME, EVENT_MANIPULATION, EVENT_OBJECT_TABLE,
       ACTION_TIMING, DEFINER
FROM information_schema.TRIGGERS
WHERE TRIGGER_SCHEMA = 'mydb'
ORDER BY EVENT_OBJECT_TABLE, ACTION_TIMING;
```

## Using BEFORE and AFTER Triggers

```sql
-- BEFORE INSERT trigger to normalize data
DELIMITER $$
CREATE TRIGGER normalize_email
BEFORE INSERT ON users
FOR EACH ROW
BEGIN
  SET NEW.email = LOWER(TRIM(NEW.email));
END$$
DELIMITER ;

-- AFTER DELETE trigger to cascade a custom cleanup
DELIMITER $$
CREATE TRIGGER cleanup_user_sessions
AFTER DELETE ON users
FOR EACH ROW
BEGIN
  DELETE FROM sessions WHERE user_id = OLD.id;
END$$
DELIMITER ;
```

## Dropping and Recreating a Trigger

MySQL does not support `ALTER TRIGGER`. To modify a trigger:

```sql
DROP TRIGGER IF EXISTS log_user_update;

DELIMITER $$
CREATE TRIGGER log_user_update
AFTER UPDATE ON users
FOR EACH ROW
BEGIN
  INSERT INTO audit_log (table_name, record_id, old_email, new_email, changed_at)
  VALUES ('users', NEW.id, OLD.email, NEW.email, NOW());
END$$
DELIMITER ;
```

## Exporting Triggers with mysqldump

```bash
# Triggers are included by default in mysqldump
mysqldump mydb users > users_with_triggers.sql

# To exclude triggers
mysqldump --skip-triggers mydb users > users_no_triggers.sql
```

## Summary

`SHOW CREATE TRIGGER` gives you the exact DDL needed to recreate any trigger in your MySQL database. Use it alongside `SHOW TRIGGERS` and `information_schema.TRIGGERS` to get a full inventory of trigger logic. Always review trigger definitions when migrating databases, as triggers can have significant side effects on data integrity and performance.
