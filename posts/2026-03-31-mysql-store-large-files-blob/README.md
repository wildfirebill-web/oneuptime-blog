# How to Store Large Files in MySQL Using BLOB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, BLOB, Binary Data, File Storage, Database

Description: Learn how to store large files in MySQL using BLOB columns, including choosing the right BLOB type and best practices for retrieval.

---

## When to Store Files in MySQL

Storing files directly in MySQL centralizes data management, simplifies backups, and ensures transactional consistency. It works well for files that are tightly coupled to database records, such as profile images or signed PDF documents. For very large files or high-volume serving, a dedicated file store or object storage may be more appropriate.

## MySQL BLOB Types

MySQL provides four BLOB types with different size limits:

| Type       | Maximum Size      |
|------------|-------------------|
| TINYBLOB   | 255 bytes         |
| BLOB       | 65,535 bytes (~64 KB) |
| MEDIUMBLOB | 16,777,215 bytes (~16 MB) |
| LONGBLOB   | 4,294,967,295 bytes (~4 GB) |

Choose the smallest type that fits your data.

## Creating a Table with a BLOB Column

```sql
CREATE TABLE documents (
  id INT AUTO_INCREMENT PRIMARY KEY,
  filename VARCHAR(255) NOT NULL,
  mime_type VARCHAR(100) NOT NULL,
  file_data MEDIUMBLOB NOT NULL,
  uploaded_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

## Inserting a File from the Filesystem

Use `LOAD_FILE()` to read a file into a BLOB column. The file must be readable by the MySQL server process:

```sql
INSERT INTO documents (filename, mime_type, file_data)
VALUES (
  'report.pdf',
  'application/pdf',
  LOAD_FILE('/tmp/report.pdf')
);
```

Note: `LOAD_FILE()` requires the `FILE` privilege and the `secure_file_priv` system variable must allow the path.

## Inserting Binary Data from an Application

Most application drivers handle binary binding directly:

```python
import mysql.connector

conn = mysql.connector.connect(
    host='localhost', user='app', password='secret', database='mydb'
)

with open('report.pdf', 'rb') as f:
    file_data = f.read()

cursor = conn.cursor()
cursor.execute(
    "INSERT INTO documents (filename, mime_type, file_data) VALUES (%s, %s, %s)",
    ('report.pdf', 'application/pdf', file_data)
)
conn.commit()
```

## Retrieving a File

```sql
SELECT filename, mime_type, file_data
FROM documents
WHERE id = 1;
```

To export the file back to disk:

```sql
SELECT file_data
INTO DUMPFILE '/tmp/retrieved_report.pdf'
FROM documents
WHERE id = 1;
```

## Performance Considerations

- Set `max_allowed_packet` to a value larger than your biggest file to avoid truncation errors.
- Avoid selecting BLOB columns in broad queries. Use a separate query to fetch the file after selecting metadata.
- Consider separating file metadata into one table and file data into another for better caching.

```sql
-- Adjust max_allowed_packet for large files
SET GLOBAL max_allowed_packet = 64 * 1024 * 1024;  -- 64 MB
```

## Summary

MySQL BLOB types let you store binary files directly in the database, ensuring consistency with related records. Choose `MEDIUMBLOB` or `LONGBLOB` for larger files, and always handle binary data through parameterized queries in your application driver. Be mindful of `max_allowed_packet` and avoid selecting BLOB columns unnecessarily in list queries.
