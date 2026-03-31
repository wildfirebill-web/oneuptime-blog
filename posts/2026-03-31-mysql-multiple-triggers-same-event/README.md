# How to Use Multiple Triggers on the Same Event in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Trigger, Multiple Trigger, FOLLOWS, PRECEDES

Description: Learn how MySQL supports multiple triggers on the same table event and how to control their execution order using FOLLOWS and PRECEDES.

---

MySQL 5.7.2 and later allow multiple triggers of the same type (e.g., multiple `AFTER INSERT` triggers) on the same table. This feature lets different teams or modules add independent trigger logic without merging everything into a single large trigger body.

## Creating Two Triggers on the Same Event

```sql
DELIMITER //

-- First trigger: write to audit log
CREATE TRIGGER trg_orders_audit
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (table_name, record_id, action, logged_at)
    VALUES ('orders', NEW.order_id, 'INSERT', NOW());
END //

-- Second trigger: update sales summary
CREATE TRIGGER trg_orders_summary
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    INSERT INTO daily_sales (sale_date, amount)
    VALUES (DATE(NEW.created_at), NEW.total)
    ON DUPLICATE KEY UPDATE amount = amount + NEW.total;
END //

DELIMITER ;
```

Both triggers fire on every `INSERT` into `orders`. Without specifying order, MySQL fires them in the order they were created.

## Controlling Execution Order

Use `FOLLOWS` or `PRECEDES` to set an explicit order relative to an existing trigger:

```sql
DELIMITER //

-- This trigger must run AFTER trg_orders_audit
CREATE TRIGGER trg_orders_notify
AFTER INSERT ON orders
FOR EACH ROW
FOLLOWS trg_orders_audit
BEGIN
    INSERT INTO notification_queue (order_id, queued_at)
    VALUES (NEW.order_id, NOW());
END //

DELIMITER ;
```

Or run it before an existing trigger:

```sql
DELIMITER //

CREATE TRIGGER trg_orders_validate
AFTER INSERT ON orders
FOR EACH ROW
PRECEDES trg_orders_audit
BEGIN
    -- runs before trg_orders_audit
    UPDATE order_stats SET insert_count = insert_count + 1;
END //

DELIMITER ;
```

## Viewing All Triggers and Their Order

```sql
SELECT TRIGGER_NAME, ACTION_ORDER, ACTION_TIMING, EVENT_MANIPULATION
FROM information_schema.TRIGGERS
WHERE TRIGGER_SCHEMA = 'mydb'
  AND EVENT_OBJECT_TABLE = 'orders'
ORDER BY ACTION_TIMING, EVENT_MANIPULATION, ACTION_ORDER;
```

`ACTION_ORDER` is a 1-based integer reflecting the execution sequence.

## Error Behavior

If any trigger in the chain raises an error (via `SIGNAL` or a runtime SQL error), MySQL stops processing remaining triggers and rolls back the triggering DML statement for InnoDB tables. Design independent triggers so failures in one do not invalidate the others when possible.

## Best Practices

- Keep each trigger small and focused on a single responsibility.
- Always specify `FOLLOWS` or `PRECEDES` when order matters to avoid relying on implicit creation order.
- Use descriptive naming conventions like `trg_{table}_{purpose}` to make the sequence readable in `information_schema`.

```sql
-- Confirm trigger order before deploying
SELECT TRIGGER_NAME, ACTION_ORDER
FROM information_schema.TRIGGERS
WHERE EVENT_OBJECT_TABLE = 'orders'
ORDER BY ACTION_ORDER;
```

## Summary

MySQL supports multiple triggers of the same type on a single table, useful for separating concerns like auditing, notifications, and aggregation. Use `FOLLOWS` or `PRECEDES` at creation time to control execution order, and always verify the final order in `information_schema.TRIGGERS`.
