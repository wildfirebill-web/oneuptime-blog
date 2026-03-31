# How to Implement a Custom Audit Trail in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Audit, Trigger

Description: Learn how to implement a custom audit trail in MySQL using triggers to log row-level INSERT, UPDATE, and DELETE operations with before and after values.

---

A custom audit trail records every data change in a database table, capturing who made the change, when it happened, and what values were before and after. Unlike generic logging, a custom audit trail is tailored to your specific compliance requirements and data model.

## Audit Trail Table Design

Create an audit table that stores change records for all audited tables:

```sql
CREATE TABLE audit_log (
  id           BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  table_name   VARCHAR(64) NOT NULL,
  operation    ENUM('INSERT', 'UPDATE', 'DELETE') NOT NULL,
  record_id    VARCHAR(100) NOT NULL,
  changed_by   VARCHAR(100) NULL,
  changed_at   DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  old_values   JSON NULL,
  new_values   JSON NULL,
  INDEX idx_audit_table_record (table_name, record_id),
  INDEX idx_audit_changed_at (changed_at)
);
```

Storing `old_values` and `new_values` as JSON provides flexibility to audit any table without modifying the audit table schema.

## Setting the Application User Context

Use a session variable to track which application user made the change:

```sql
-- Set at the start of each request/transaction
SET @current_user = 'jane.doe@example.com';
```

In application code:

```python
cursor.execute("SET @current_user = %s", (current_user.email,))
```

## Creating Audit Triggers

Create separate triggers for INSERT, UPDATE, and DELETE:

```sql
DELIMITER //

-- INSERT trigger
CREATE TRIGGER trg_orders_after_insert
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
  INSERT INTO audit_log (table_name, operation, record_id, changed_by, new_values)
  VALUES (
    'orders',
    'INSERT',
    NEW.id,
    @current_user,
    JSON_OBJECT(
      'customer_id', NEW.customer_id,
      'total',       NEW.total,
      'status',      NEW.status,
      'created_at',  NEW.created_at
    )
  );
END //

-- UPDATE trigger
CREATE TRIGGER trg_orders_after_update
AFTER UPDATE ON orders
FOR EACH ROW
BEGIN
  INSERT INTO audit_log (table_name, operation, record_id, changed_by, old_values, new_values)
  VALUES (
    'orders',
    'UPDATE',
    NEW.id,
    @current_user,
    JSON_OBJECT(
      'customer_id', OLD.customer_id,
      'total',       OLD.total,
      'status',      OLD.status
    ),
    JSON_OBJECT(
      'customer_id', NEW.customer_id,
      'total',       NEW.total,
      'status',      NEW.status
    )
  );
END //

-- DELETE trigger
CREATE TRIGGER trg_orders_after_delete
AFTER DELETE ON orders
FOR EACH ROW
BEGIN
  INSERT INTO audit_log (table_name, operation, record_id, changed_by, old_values)
  VALUES (
    'orders',
    'DELETE',
    OLD.id,
    @current_user,
    JSON_OBJECT(
      'customer_id', OLD.customer_id,
      'total',       OLD.total,
      'status',      OLD.status
    )
  );
END //

DELIMITER ;
```

## Querying the Audit Trail

Find all changes to a specific record:

```sql
SELECT operation, changed_by, changed_at, old_values, new_values
FROM audit_log
WHERE table_name = 'orders' AND record_id = '12345'
ORDER BY changed_at;
```

Find changes made by a specific user:

```sql
SELECT table_name, operation, record_id, changed_at
FROM audit_log
WHERE changed_by = 'jane.doe@example.com'
  AND changed_at > NOW() - INTERVAL 24 HOUR
ORDER BY changed_at DESC;
```

Find what changed in an UPDATE:

```sql
SELECT
  record_id,
  changed_by,
  changed_at,
  JSON_EXTRACT(old_values, '$.status') AS old_status,
  JSON_EXTRACT(new_values, '$.status') AS new_status
FROM audit_log
WHERE table_name = 'orders'
  AND operation = 'UPDATE'
  AND JSON_EXTRACT(old_values, '$.status') != JSON_EXTRACT(new_values, '$.status');
```

## Archiving Old Audit Records

Audit tables grow quickly. Archive old records to a separate table:

```sql
CREATE TABLE audit_log_archive LIKE audit_log;

INSERT INTO audit_log_archive
SELECT * FROM audit_log WHERE changed_at < NOW() - INTERVAL 90 DAY;

DELETE FROM audit_log WHERE changed_at < NOW() - INTERVAL 90 DAY;
```

## Summary

A custom MySQL audit trail uses `AFTER INSERT`, `AFTER UPDATE`, and `AFTER DELETE` triggers to write change records to an `audit_log` table. Storing before and after values as JSON provides flexibility for querying specific field changes. Session variables enable user attribution, and JSON extraction functions make it easy to query what changed and who changed it.
