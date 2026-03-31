# How to Implement Audit Trails in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Audit Trail, Triggers, Security, Compliance

Description: Learn how to implement audit trails in MySQL using triggers and audit tables to track INSERT, UPDATE, and DELETE operations on sensitive data.

---

## Why Audit Trails Matter

Audit trails record who changed what and when. They are essential for regulatory compliance (HIPAA, SOX, GDPR), fraud detection, and debugging production data issues. MySQL does not have a built-in audit log in the Community Edition (only Enterprise), but triggers provide a reliable alternative.

## Setting Up the Audit Table

Create a generic audit log table that captures changes across any audited table:

```sql
CREATE TABLE audit_log (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  table_name VARCHAR(64) NOT NULL,
  operation ENUM('INSERT', 'UPDATE', 'DELETE') NOT NULL,
  record_id BIGINT NOT NULL,
  changed_by VARCHAR(100) NOT NULL,
  changed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  old_values JSON,
  new_values JSON,
  INDEX idx_table_record (table_name, record_id),
  INDEX idx_changed_at (changed_at)
);
```

Using JSON columns for `old_values` and `new_values` allows the same audit table to serve all tables.

## Creating Triggers for an Orders Table

### Sample Orders Table

```sql
CREATE TABLE orders (
  id INT AUTO_INCREMENT PRIMARY KEY,
  customer_id INT NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  total_amount DECIMAL(12, 2) NOT NULL,
  notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### INSERT Trigger

```sql
CREATE TRIGGER orders_after_insert
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
  INSERT INTO audit_log (table_name, operation, record_id, changed_by, new_values)
  VALUES (
    'orders',
    'INSERT',
    NEW.id,
    USER(),
    JSON_OBJECT(
      'customer_id', NEW.customer_id,
      'status', NEW.status,
      'total_amount', NEW.total_amount,
      'notes', NEW.notes
    )
  );
END;
```

### UPDATE Trigger

```sql
CREATE TRIGGER orders_after_update
AFTER UPDATE ON orders
FOR EACH ROW
BEGIN
  INSERT INTO audit_log (table_name, operation, record_id, changed_by, old_values, new_values)
  VALUES (
    'orders',
    'UPDATE',
    NEW.id,
    USER(),
    JSON_OBJECT(
      'customer_id', OLD.customer_id,
      'status', OLD.status,
      'total_amount', OLD.total_amount,
      'notes', OLD.notes
    ),
    JSON_OBJECT(
      'customer_id', NEW.customer_id,
      'status', NEW.status,
      'total_amount', NEW.total_amount,
      'notes', NEW.notes
    )
  );
END;
```

### DELETE Trigger

```sql
CREATE TRIGGER orders_after_delete
AFTER DELETE ON orders
FOR EACH ROW
BEGIN
  INSERT INTO audit_log (table_name, operation, record_id, changed_by, old_values)
  VALUES (
    'orders',
    'DELETE',
    OLD.id,
    USER(),
    JSON_OBJECT(
      'customer_id', OLD.customer_id,
      'status', OLD.status,
      'total_amount', OLD.total_amount,
      'notes', OLD.notes
    )
  );
END;
```

## Testing the Audit Trail

```sql
INSERT INTO orders (customer_id, status, total_amount) VALUES (1, 'pending', 99.99);

UPDATE orders SET status = 'shipped' WHERE id = 1;

DELETE FROM orders WHERE id = 1;

SELECT * FROM audit_log ORDER BY id;
```

## Querying the Audit Trail

### Who Changed a Specific Record

```sql
SELECT operation, changed_by, changed_at, old_values, new_values
FROM audit_log
WHERE table_name = 'orders' AND record_id = 1
ORDER BY changed_at;
```

### Recent Status Changes

```sql
SELECT
  record_id,
  changed_by,
  changed_at,
  JSON_UNQUOTE(JSON_EXTRACT(old_values, '$.status')) AS old_status,
  JSON_UNQUOTE(JSON_EXTRACT(new_values, '$.status')) AS new_status
FROM audit_log
WHERE table_name = 'orders'
  AND operation = 'UPDATE'
  AND JSON_EXTRACT(old_values, '$.status') != JSON_EXTRACT(new_values, '$.status')
ORDER BY changed_at DESC
LIMIT 20;
```

## Tracking Application User vs DB User

If multiple app users share the same DB connection user, pass the application user via a session variable:

```sql
SET @app_user = 'john.doe@example.com';
```

Then in the trigger, use `COALESCE(@app_user, USER())` as the `changed_by` value.

## Archiving Old Audit Records

Keep the audit log manageable by archiving old records:

```sql
-- Archive records older than 1 year to a cold storage table
INSERT INTO audit_log_archive SELECT * FROM audit_log WHERE changed_at < NOW() - INTERVAL 1 YEAR;
DELETE FROM audit_log WHERE changed_at < NOW() - INTERVAL 1 YEAR;
```

## Summary

Implementing audit trails in MySQL using AFTER triggers and a JSON-based audit log table provides a flexible, compliant record of all data changes. By capturing old and new values as JSON, a single audit table can serve the entire application. Combine this with session variables for application-level user tracking, and schedule periodic archiving to keep audit logs performant.
