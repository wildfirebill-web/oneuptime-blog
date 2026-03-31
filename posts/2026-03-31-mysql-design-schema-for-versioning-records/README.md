# How to Design a Schema for Versioning Records in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Versioning, Schema Design, Audit

Description: Learn how to version database records in MySQL using append-only history tables and current-version pointers with practical query examples.

---

Record versioning stores every state a row has ever been in, enabling point-in-time queries, diff comparisons, and rollback. There are two main approaches: a separate history table or a version-column on the same table.

## Approach 1 - Separate History Table

Keep the live row in the main table and archive every change to a history table.

```sql
CREATE TABLE documents (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT,
    title      VARCHAR(255) NOT NULL,
    body       TEXT         NOT NULL,
    version    INT UNSIGNED NOT NULL DEFAULT 1,
    updated_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    updated_by INT UNSIGNED NULL,
    PRIMARY KEY (id)
);

CREATE TABLE document_history (
    history_id  BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    document_id INT UNSIGNED    NOT NULL,
    title       VARCHAR(255)    NOT NULL,
    body        TEXT            NOT NULL,
    version     INT UNSIGNED    NOT NULL,
    saved_at    DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    saved_by    INT UNSIGNED    NULL,
    PRIMARY KEY (history_id),
    KEY idx_doc_version (document_id, version)
);
```

## Writing Versions With a Trigger

```sql
DELIMITER $$
CREATE TRIGGER trg_document_before_update
BEFORE UPDATE ON documents
FOR EACH ROW
BEGIN
    INSERT INTO document_history
        (document_id, title, body, version, saved_by)
    VALUES
        (OLD.id, OLD.title, OLD.body, OLD.version, OLD.updated_by);
    SET NEW.version = OLD.version + 1;
END$$
DELIMITER ;
```

## Querying Version History

```sql
-- Full version history of a document
SELECT version, title, saved_at, saved_by
FROM   document_history
WHERE  document_id = 5
ORDER BY version DESC;

-- Retrieve a specific version
SELECT title, body
FROM   document_history
WHERE  document_id = 5
AND    version     = 3;
```

## Approach 2 - Versioned Rows in a Single Table

All versions live in one table with a composite key:

```sql
CREATE TABLE document_versions (
    document_id INT UNSIGNED NOT NULL,
    version     INT UNSIGNED NOT NULL,
    title       VARCHAR(255) NOT NULL,
    body        TEXT         NOT NULL,
    is_current  TINYINT(1)   NOT NULL DEFAULT 0,
    created_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by  INT UNSIGNED NULL,
    PRIMARY KEY (document_id, version),
    KEY idx_current (document_id, is_current)
);
```

## Marking the Latest Version

```sql
START TRANSACTION;

-- Demote old current version
UPDATE document_versions
SET is_current = 0
WHERE document_id = 5 AND is_current = 1;

-- Insert the new version
INSERT INTO document_versions (document_id, version, title, body, is_current, created_by)
SELECT 5, COALESCE(MAX(version), 0) + 1, 'New Title', 'New body...', 1, 42
FROM   document_versions WHERE document_id = 5;

COMMIT;
```

## Fetching the Current Version

```sql
SELECT title, body, version, created_at
FROM   document_versions
WHERE  document_id = 5
AND    is_current  = 1;
```

## Rolling Back to a Previous Version

```sql
START TRANSACTION;
UPDATE document_versions SET is_current = 0 WHERE document_id = 5;
UPDATE document_versions SET is_current = 1 WHERE document_id = 5 AND version = 2;
COMMIT;
```

## Summary

Use a separate history table with BEFORE UPDATE triggers for transparent versioning without application changes. Use a versioned rows table when you need random access to any version without joining two tables. Always include an index on `(entity_id, version)` and keep writes inside transactions to prevent partial version states.
