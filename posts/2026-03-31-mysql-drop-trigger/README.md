# How to Drop a Trigger in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Trigger, DDL, Database Management, DROP

Description: Learn how to drop a MySQL trigger safely using DROP TRIGGER IF EXISTS, check which triggers exist, and understand what happens when a trigger is removed.

---

Dropping a trigger in MySQL removes its definition from the database permanently. The operation is immediate, requires no downtime, and takes effect for all subsequent DML statements on the associated table.

## Basic DROP TRIGGER Syntax

```sql
DROP TRIGGER trigger_name;
```

To drop a trigger in a specific database, qualify the name with the schema:

```sql
DROP TRIGGER mydb.trg_orders_after_insert;
```

If the trigger does not exist, MySQL raises an error. Use `IF EXISTS` to suppress it:

```sql
DROP TRIGGER IF EXISTS trg_orders_after_insert;
```

## Checking Whether a Trigger Exists

Before dropping, confirm the trigger exists and note its table and event:

```sql
SHOW TRIGGERS LIKE 'orders';
```

Or query `information_schema`:

```sql
SELECT TRIGGER_NAME, EVENT_MANIPULATION, EVENT_OBJECT_TABLE, ACTION_TIMING
FROM information_schema.TRIGGERS
WHERE TRIGGER_SCHEMA = 'mydb'
  AND TRIGGER_NAME = 'trg_orders_after_insert';
```

An empty result means the trigger does not exist.

## Required Privileges

To drop a trigger you need the `TRIGGER` privilege on the table it is associated with:

```sql
GRANT TRIGGER ON mydb.orders TO 'developer'@'localhost';
```

Verify current privileges:

```sql
SHOW GRANTS FOR CURRENT_USER();
```

## Dropping All Triggers on a Table

MySQL does not have a single command to drop all triggers on a table at once. Generate the `DROP TRIGGER` statements using a query and execute them:

```sql
SELECT CONCAT('DROP TRIGGER IF EXISTS ', TRIGGER_SCHEMA, '.', TRIGGER_NAME, ';') AS stmt
FROM information_schema.TRIGGERS
WHERE TRIGGER_SCHEMA = 'mydb'
  AND EVENT_OBJECT_TABLE = 'orders';
```

Copy and run the generated statements, or use a prepared statement in a stored procedure:

```sql
DELIMITER //

CREATE PROCEDURE drop_all_triggers_on_table(IN p_db VARCHAR(64), IN p_table VARCHAR(64))
BEGIN
    DECLARE v_name  VARCHAR(64);
    DECLARE done    INT DEFAULT 0;
    DECLARE cur CURSOR FOR
        SELECT TRIGGER_NAME FROM information_schema.TRIGGERS
        WHERE TRIGGER_SCHEMA = p_db AND EVENT_OBJECT_TABLE = p_table;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

    OPEN cur;
    read_loop: LOOP
        FETCH cur INTO v_name;
        IF done THEN LEAVE read_loop; END IF;
        SET @sql = CONCAT('DROP TRIGGER IF EXISTS `', p_db, '`.`', v_name, '`');
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;
    END LOOP;
    CLOSE cur;
END //

DELIMITER ;
```

## What Happens When a Trigger Is Dropped

- All future DML on the table executes without the trigger firing.
- Rows already modified are not affected retroactively.
- Any objects that depend on the trigger (none typically do in MySQL) are unaffected.
- The trigger body is permanently deleted and cannot be recovered from MySQL metadata alone - keep source-controlled `.sql` files for all triggers.

## Summary

Use `DROP TRIGGER IF EXISTS` for safe, error-free trigger removal. Always check `SHOW TRIGGERS` or `information_schema.TRIGGERS` first to confirm the trigger name and associated table, and store trigger definitions in version control so you can recreate them if needed.
