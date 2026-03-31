# How to Use MEDIUMTEXT Data Type in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Database, Data Type, Text, Storage

Description: Learn when and how to use the MEDIUMTEXT data type in MySQL to store large text up to 16 MB, with examples and indexing strategies.

---

## What Is the MEDIUMTEXT Data Type

`MEDIUMTEXT` is a large-object text type in MySQL that can store up to 16,777,215 bytes (16 MB). It sits between `TEXT` (64 KB) and `LONGTEXT` (4 GB) in the TEXT family.

Like all TEXT types, `MEDIUMTEXT` stores data off-page - outside the main row buffer - so it does not count toward the 65,535-byte row-size limit.

## Declaring a MEDIUMTEXT Column

```sql
CREATE TABLE documents (
    id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title      VARCHAR(300) NOT NULL,
    content    MEDIUMTEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

## Inserting and Retrieving Large Text

```sql
INSERT INTO documents (title, content)
VALUES ('Annual Report 2025', LOAD_FILE('/tmp/report2025.txt'));

SELECT id, title, LENGTH(content) AS content_bytes FROM documents;
```

## Byte and Character Limits

With `utf8mb4`, the 16,777,215-byte limit equals up to 4,194,303 four-byte characters. For ASCII content the full 16 MB is available.

```sql
SELECT CHARACTER_MAXIMUM_LENGTH
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'documents'
  AND COLUMN_NAME = 'content';
-- Returns: 4194303 (for utf8mb4)
```

## Full-Text Indexing

`MEDIUMTEXT` supports full-text indexes, which are the recommended way to search large text columns:

```sql
ALTER TABLE documents ADD FULLTEXT INDEX ft_content (content);

SELECT id, title
FROM documents
WHERE MATCH(content) AGAINST ('+revenue +growth' IN BOOLEAN MODE)
ORDER BY MATCH(content) AGAINST ('+revenue +growth' IN BOOLEAN MODE) DESC
LIMIT 10;
```

## Prefix Indexes for Sorting and Grouping

Direct indexes are not allowed on text columns without a prefix length. For ordered queries, use a prefix or a generated hash column:

```sql
-- Prefix index for partial matching
ALTER TABLE documents ADD INDEX idx_content_prefix (content(200));

-- Alternative: store a hash for exact-match lookups
ALTER TABLE documents
    ADD COLUMN content_hash CHAR(64) GENERATED ALWAYS AS (SHA2(content, 256)) STORED,
    ADD INDEX idx_hash (content_hash);
```

## Practical Use Cases

```sql
-- Storing HTML email templates
CREATE TABLE email_templates (
    id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name     VARCHAR(100) NOT NULL UNIQUE,
    html     MEDIUMTEXT NOT NULL,
    text_alt MEDIUMTEXT
);

-- Storing exported SQL dumps for audit purposes
CREATE TABLE schema_snapshots (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    captured_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    dump_sql    MEDIUMTEXT NOT NULL
);
```

## Choosing MEDIUMTEXT vs TEXT vs LONGTEXT

| Type | Max Size | Use When |
|---|---|---|
| TEXT | 64 KB | Articles, comments, short logs |
| MEDIUMTEXT | 16 MB | Long documents, HTML templates, reports |
| LONGTEXT | 4 GB | Very large exports, base64-encoded files |

Choose `MEDIUMTEXT` when content can grow beyond 64 KB but 16 MB is a reasonable upper bound.

## Summary

`MEDIUMTEXT` handles large text payloads up to 16 MB without affecting the main row size. It is well-suited for rich document content, long log entries, and HTML templates. Use full-text indexes for search and prefix indexes when you need partial-text ordering. If your content can be reliably bounded under 64 KB, `TEXT` is sufficient.
