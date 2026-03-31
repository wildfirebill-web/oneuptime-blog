# How to Implement Audit Logging with MySQL Triggers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Trigger, Audit, Logging, Compliance

Description: Learn how to build a practical audit logging system in MySQL using AFTER INSERT, AFTER UPDATE, and AFTER DELETE triggers to track all data changes.

---

Audit logging records who changed what data and when. MySQL triggers are an effective way to implement this at the database layer, ensuring that every INSERT, UPDATE, and DELETE is captured regardless of which application or tool caused the change.

## Designing the Audit Table

A flexible audit table stores the table name, the type of action, the changed row's primary key, old and new values (as JSON), the user, and the timestamp:

```sql
CREATE TABLE audit_log (
    id           BIGINT AUTO_INCREMENT PRIMARY KEY,
    table_name   VARCHAR(64)  NOT NULL,
    record_id    BIGINT       NOT NULL,
    action       ENUM('INSERT','UPDATE','DELETE') NOT NULL,
    old_values   JSON,
    new_values   JSON,
    changed_by   VARCHAR(100) NOT NULL,
    changed_at   DATETIME(3)  NOT NULL DEFAULT NOW(3),
    INDEX idx_table_record (table_name, record_id),
    INDEX idx_changed_at (changed_at)
);
```

## AFTER INSERT Trigger

```sql
DELIMITER //

CREATE TRIGGER trg_customers_after_insert
AFTER INSERT ON customers
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (table_name, record_id, action, new_values, changed_by, changed_at)
    VALUES (
        'customers',
        NEW.customer_id,
        'INSERT',
        JSON_OBJECT(
            'name',  NEW.name,
            'email', NEW.email,
            'status', NEW.status
        ),
        USER(),
        NOW(3)
    );
END //

DELIMITER ;
```

## AFTER UPDATE Trigger

Record both the old state and the new state:

```sql
DELIMITER //

CREATE TRIGGER trg_customers_after_update
AFTER UPDATE ON customers
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (table_name, record_id, action, old_values, new_values, changed_by, changed_at)
    VALUES (
        'customers',
        NEW.customer_id,
        'UPDATE',
        JSON_OBJECT('name', OLD.name, 'email', OLD.email, 'status', OLD.status),
        JSON_OBJECT('name', NEW.name, 'email', NEW.email, 'status', NEW.status),
        USER(),
        NOW(3)
    );
END //

DELIMITER ;
```

## AFTER DELETE Trigger

Capture the deleted row in `old_values`:

```sql
DELIMITER //

CREATE TRIGGER trg_customers_after_delete
AFTER DELETE ON customers
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (table_name, record_id, action, old_values, changed_by, changed_at)
    VALUES (
        'customers',
        OLD.customer_id,
        'DELETE',
        JSON_OBJECT('name', OLD.name, 'email', OLD.email, 'status', OLD.status),
        USER(),
        NOW(3)
    );
END //

DELIMITER ;
```

## Querying the Audit Log

Find all changes to a specific customer:

```sql
SELECT action, old_values, new_values, changed_by, changed_at
FROM audit_log
WHERE table_name = 'customers'
  AND record_id = 42
ORDER BY changed_at DESC;
```

Find all changes made today:

```sql
SELECT table_name, record_id, action, changed_by, changed_at
FROM audit_log
WHERE changed_at >= CURDATE()
ORDER BY changed_at DESC;
```

## Performance Considerations

Every DML statement on an audited table adds one `INSERT` into `audit_log`. For high-throughput tables, consider:
- Using an `INSERT DELAYED` equivalent (insert into a queue table and flush asynchronously)
- Partitioning `audit_log` by `changed_at` to keep recent data fast to query
- Archiving old audit rows to cold storage periodically with a scheduled event

## Summary

Implement audit logging by creating AFTER INSERT, AFTER UPDATE, and AFTER DELETE triggers that write `NEW` and `OLD` row data as JSON into a central `audit_log` table. This approach captures all changes at the database layer regardless of the source, and the JSON format keeps the schema flexible across multiple audited tables.
