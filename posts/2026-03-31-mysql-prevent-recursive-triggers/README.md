# How to Prevent Recursive Triggers in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Trigger, Recursive, innodb_recursive_triggers, Configuration

Description: Learn why MySQL triggers can become recursive, how the innodb_recursive_triggers variable controls this, and how to write triggers that avoid infinite loops.

---

A recursive trigger occurs when a trigger fires, and the SQL inside the trigger body modifies the same table that the trigger is attached to, causing the trigger to fire again. Without a guard, this creates an infinite loop that eventually results in a "max recursion depth exceeded" error or, if recursion is disabled, silently does nothing.

## How MySQL Handles Trigger Recursion

MySQL controls recursion with the `innodb_recursive_triggers` variable (effectively controlled by the session variable `@@innodb_lock_wait_timeout` indirectly, but the actual recursion guard is the internal MySQL setting). The relevant behavior is:

```sql
-- Check the current setting
SHOW VARIABLES LIKE 'innodb_recursive_triggers';
```

By default MySQL **does not allow** a trigger to fire another trigger of the same type on the same table. If a `BEFORE UPDATE` trigger updates the same table, the nested `BEFORE UPDATE` trigger does not fire.

However, indirect recursion can still occur: a trigger on table A modifies table B, and a trigger on table B modifies table A.

## Direct Recursion Example

```sql
DELIMITER //

-- This trigger updates the same table - the nested trigger will NOT fire by default
CREATE TRIGGER trg_employees_after_update
AFTER UPDATE ON employees
FOR EACH ROW
BEGIN
    -- This UPDATE on employees does NOT re-fire trg_employees_after_update
    UPDATE employees SET last_modified = NOW() WHERE emp_id = NEW.emp_id;
END //

DELIMITER ;
```

While the inner `UPDATE` does run, it does not recursively re-trigger because MySQL's default behavior suppresses it. The `last_modified` column will be updated, but the trigger will not fire a second time.

## Indirect Recursion - The Real Risk

```sql
-- Trigger on table_a updates table_b
CREATE TRIGGER trg_a_after_update
AFTER UPDATE ON table_a
FOR EACH ROW
BEGIN
    UPDATE table_b SET synced_val = NEW.value WHERE id = NEW.id;
END;

-- Trigger on table_b updates table_a - INFINITE LOOP
CREATE TRIGGER trg_b_after_update
AFTER UPDATE ON table_b
FOR EACH ROW
BEGIN
    UPDATE table_a SET mirrored_val = NEW.synced_val WHERE id = NEW.id;
END;
```

MySQL reports:

```text
ERROR 1436 (HY000): Thread stack overrun
```

## Prevention Strategy 1 - Guard with a Conditional Check

Only update the other table if the value actually changed:

```sql
DELIMITER //

CREATE TRIGGER trg_b_after_update
AFTER UPDATE ON table_b
FOR EACH ROW
BEGIN
    IF OLD.synced_val <> NEW.synced_val THEN
        UPDATE table_a SET mirrored_val = NEW.synced_val WHERE id = NEW.id;
    END IF;
END //

DELIMITER ;
```

When `table_a` is updated, it sets `table_b.synced_val = NEW.value`. When `table_b` fires its trigger, `OLD.synced_val` equals `NEW.synced_val` (no real change) so the `UPDATE table_a` is skipped.

## Prevention Strategy 2 - Use a Session Variable as a Flag

```sql
DELIMITER //

CREATE TRIGGER trg_a_after_update
AFTER UPDATE ON table_a
FOR EACH ROW
BEGIN
    IF @trigger_running IS NULL THEN
        SET @trigger_running = 1;
        UPDATE table_b SET synced_val = NEW.value WHERE id = NEW.id;
        SET @trigger_running = NULL;
    END IF;
END //

DELIMITER ;
```

The session variable `@trigger_running` acts as a reentrance guard. Set it to `NULL` when done so subsequent legitimate calls work correctly.

## Summary

MySQL suppresses direct trigger recursion by default, but indirect recursion across two tables can still cause stack overflow errors. Prevent this by adding value-change guards (`IF OLD.col <> NEW.col`) or session variable flags to break the cycle and ensure triggers only fire when a genuine data change occurs.
