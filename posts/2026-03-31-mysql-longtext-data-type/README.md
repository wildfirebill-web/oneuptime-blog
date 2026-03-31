# How to Use LONGTEXT Data Type in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Database, Data Type, Text, Storage

Description: Learn how to use LONGTEXT in MySQL to store up to 4 GB of character data, with practical examples and performance considerations.

---

## What Is the LONGTEXT Data Type

`LONGTEXT` is the largest text type in MySQL, capable of storing up to 4,294,967,295 bytes (4 GB) of character data. It is the top of the TEXT hierarchy: `TINYTEXT`, `TEXT`, `MEDIUMTEXT`, and `LONGTEXT`.

Like all TEXT types, `LONGTEXT` stores data off-page and does not count toward the 65,535-byte per-row limit.

## Declaring a LONGTEXT Column

```sql
CREATE TABLE raw_logs (
    id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    source     VARCHAR(100) NOT NULL,
    log_data   LONGTEXT NOT NULL,
    captured_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

## Inserting LONGTEXT Data

```sql
-- Insert from a file using LOAD_FILE (requires FILE privilege and secure_file_priv)
INSERT INTO raw_logs (source, log_data)
VALUES ('app-server-01', LOAD_FILE('/var/log/app/app.log'));

-- Insert programmatically from application code (Python example)
```

```python
import mysql.connector

conn = mysql.connector.connect(host='localhost', user='root', database='mydb')
cursor = conn.cursor()

with open('/var/log/app/app.log', 'r') as f:
    log_content = f.read()

cursor.execute(
    "INSERT INTO raw_logs (source, log_data) VALUES (%s, %s)",
    ('app-server-01', log_content)
)
conn.commit()
```

## Querying LONGTEXT

Because full scans of `LONGTEXT` columns can be very slow, fetch only what you need:

```sql
-- Get the first 500 characters as a preview
SELECT id, source, LEFT(log_data, 500) AS preview FROM raw_logs;

-- Check total size of stored log data
SELECT source, SUM(LENGTH(log_data)) AS total_bytes
FROM raw_logs
GROUP BY source;
```

## Full-Text Search on LONGTEXT

`LONGTEXT` supports full-text indexes. Be aware that building a full-text index over 4 GB columns can be resource-intensive:

```sql
ALTER TABLE raw_logs ADD FULLTEXT INDEX ft_log (log_data);

SELECT id, source
FROM raw_logs
WHERE MATCH(log_data) AGAINST ('OutOfMemoryError' IN BOOLEAN MODE)
ORDER BY captured_at DESC
LIMIT 20;
```

## Performance Considerations

- **Network transfer**: Selecting large `LONGTEXT` columns transfers all bytes over the connection. Always use `LEFT()`, `SUBSTRING()`, or full-text search instead of `SELECT *`.
- **Sorting**: Sorting on `LONGTEXT` forces an on-disk temporary table. Use a separate indexed column (e.g., a hash or timestamp) for ordering.
- **Backup size**: Large `LONGTEXT` columns inflate `mysqldump` output significantly. Consider storing very large payloads in object storage and keeping only a reference URL in MySQL.

```sql
-- Pattern: store reference URL instead of giant blob
CREATE TABLE reports (
    id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name       VARCHAR(200) NOT NULL,
    storage_url VARCHAR(500),
    summary    MEDIUMTEXT
);
```

## Choosing the Right Text Type

| Type | Max Size | Typical Use |
|---|---|---|
| TINYTEXT | 255 bytes | Short hints, labels |
| TEXT | 64 KB | Articles, comments |
| MEDIUMTEXT | 16 MB | Documents, HTML templates |
| LONGTEXT | 4 GB | Raw logs, base64 blobs, large exports |

## Summary

`LONGTEXT` is the right choice when you genuinely need to store multi-megabyte or gigabyte-scale text in MySQL. In practice, it is most common for raw log archiving, large XML or JSON exports, and base64-encoded file content. For most application text fields, `TEXT` or `MEDIUMTEXT` is sufficient and more performant.
