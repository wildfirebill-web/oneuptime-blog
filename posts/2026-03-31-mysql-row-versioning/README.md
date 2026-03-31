# How to Implement Row Versioning in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Audit, Schema, History, Trigger

Description: Learn how to implement row versioning in MySQL to track the full history of record changes with timestamps and version numbers.

---

## What Is Row Versioning?

Row versioning maintains a complete history of changes to a record. Rather than overwriting data, each update creates a new version of the row. This is essential for audit trails, time-travel queries, and regulatory compliance.

## Schema Design: Versioned Table

The simplest approach adds version tracking columns to the main table alongside a separate history table:

```sql
-- Main table: holds only the current version
CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    description TEXT,
    version INT NOT NULL DEFAULT 1,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    updated_by VARCHAR(100)
);

-- History table: holds all previous versions
CREATE TABLE products_history (
    history_id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    description TEXT,
    version INT NOT NULL,
    changed_at TIMESTAMP NOT NULL,
    changed_by VARCHAR(100),
    operation ENUM('INSERT', 'UPDATE', 'DELETE') NOT NULL,
    INDEX idx_product_version (product_id, version),
    INDEX idx_product_changed (product_id, changed_at)
);
```

## Automating Versioning with Triggers

```sql
DELIMITER $$

-- Capture history on UPDATE
CREATE TRIGGER products_after_update
AFTER UPDATE ON products
FOR EACH ROW
BEGIN
    INSERT INTO products_history
        (product_id, name, price, description, version, changed_at, changed_by, operation)
    VALUES
        (OLD.id, OLD.name, OLD.price, OLD.description,
         OLD.version, OLD.updated_at, OLD.updated_by, 'UPDATE');
END$$

-- Capture history on DELETE
CREATE TRIGGER products_after_delete
AFTER DELETE ON products
FOR EACH ROW
BEGIN
    INSERT INTO products_history
        (product_id, name, price, description, version, changed_at, changed_by, operation)
    VALUES
        (OLD.id, OLD.name, OLD.price, OLD.description,
         OLD.version, NOW(), OLD.updated_by, 'DELETE');
END$$

DELIMITER ;
```

## Application-Level Version Increment

```sql
-- Update with version increment
UPDATE products
SET name = 'New Product Name',
    price = 29.99,
    version = version + 1,
    updated_by = 'admin'
WHERE id = 1;
-- Trigger fires, saving OLD version to history
```

## Querying Version History

```sql
-- View full change history for a product
SELECT
    h.version,
    h.name,
    h.price,
    h.changed_at,
    h.changed_by,
    h.operation
FROM products_history h
WHERE h.product_id = 1
ORDER BY h.version;

-- View the record as it was at a specific point in time
SELECT *
FROM products_history
WHERE product_id = 1
  AND changed_at <= '2025-06-01 12:00:00'
ORDER BY changed_at DESC
LIMIT 1;
```

## Alternative: Temporal Table Pattern

A fully temporal approach stores all versions in one table:

```sql
CREATE TABLE products_temporal (
    id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    valid_from TIMESTAMP NOT NULL,
    valid_to TIMESTAMP NULL DEFAULT NULL,
    is_current TINYINT(1) NOT NULL DEFAULT 1,
    PRIMARY KEY (id, valid_from),
    INDEX idx_current (id, is_current)
);

-- Insert new version
UPDATE products_temporal
SET valid_to = CURRENT_TIMESTAMP, is_current = 0
WHERE id = 1 AND is_current = 1;

INSERT INTO products_temporal (id, name, price, valid_from, is_current)
VALUES (1, 'Updated Name', 29.99, CURRENT_TIMESTAMP, 1);
```

## Restoring a Previous Version

```sql
-- Restore product to a specific version
INSERT INTO products (id, name, price, description, version, updated_by)
SELECT
    product_id,
    name,
    price,
    description,
    (SELECT MAX(version) FROM products WHERE id = 1) + 1,
    'restore_script'
FROM products_history
WHERE product_id = 1
  AND version = 3
ON DUPLICATE KEY UPDATE
    name = VALUES(name),
    price = VALUES(price),
    description = VALUES(description),
    version = VALUES(version),
    updated_by = VALUES(updated_by);
```

## Summary

Row versioning in MySQL combines a history table with triggers to capture every state a record has been in. This enables time-travel queries, audit compliance, and data recovery without complex application-layer logic. The trigger-based approach is transparent to application code, capturing changes automatically whenever rows are modified or deleted.
