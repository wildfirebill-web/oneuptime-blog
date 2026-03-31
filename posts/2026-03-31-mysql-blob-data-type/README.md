# How to Use BLOB Data Type in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Database, Data Type, Binary, Storage

Description: Learn how the BLOB data type works in MySQL for storing binary data up to 64 KB, with practical examples for images, files, and encrypted content.

---

## What Is the BLOB Data Type

`BLOB` (Binary Large OBject) stores binary data up to 65,535 bytes (64 KB). It is the binary counterpart of the `TEXT` type and is part of the BLOB family: `TINYBLOB`, `BLOB`, `MEDIUMBLOB`, and `LONGBLOB`.

`BLOB` data is stored off-page, so large values do not inflate the main row size and do not count toward the 65,535-byte row limit.

## Declaring a BLOB Column

```sql
CREATE TABLE profile_images (
    user_id    INT UNSIGNED PRIMARY KEY,
    image_data BLOB NOT NULL,
    mime_type  VARCHAR(50) NOT NULL,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

## Inserting BLOB Data

```sql
-- Insert using LOAD_FILE (requires FILE privilege and secure_file_priv setting)
INSERT INTO profile_images (user_id, image_data, mime_type)
VALUES (1, LOAD_FILE('/tmp/avatar.jpg'), 'image/jpeg');
```

```python
import mysql.connector

conn = mysql.connector.connect(host='localhost', user='root', database='mydb')
cursor = conn.cursor()

with open('avatar.jpg', 'rb') as f:
    image_bytes = f.read()

cursor.execute(
    "INSERT INTO profile_images (user_id, image_data, mime_type) VALUES (%s, %s, %s)",
    (1, image_bytes, 'image/jpeg')
)
conn.commit()
```

## Retrieving BLOB Data

```sql
SELECT user_id, mime_type, LENGTH(image_data) AS size_bytes FROM profile_images;
```

```python
cursor.execute(
    "SELECT image_data, mime_type FROM profile_images WHERE user_id = %s", (1,)
)
row = cursor.fetchone()
image_bytes, mime_type = row

with open('retrieved_avatar.jpg', 'wb') as f:
    f.write(bytes(image_bytes))
```

## Storing Encrypted Content

```sql
CREATE TABLE encrypted_docs (
    id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title    VARCHAR(200) NOT NULL,
    content  BLOB NOT NULL
);

-- Encrypt before storing
INSERT INTO encrypted_docs (title, content)
VALUES ('Contract', AES_ENCRYPT('Confidential contract text...', UNHEX(SHA2('key', 256))));

-- Decrypt on retrieval
SELECT id, title, CAST(AES_DECRYPT(content, UNHEX(SHA2('key', 256))) AS CHAR) AS plaintext
FROM encrypted_docs
WHERE id = 1;
```

## BLOB Best Practices

**Do not select BLOB columns unnecessarily:**

```sql
-- Good: only fetch the BLOB when needed
SELECT user_id, mime_type, LENGTH(image_data) FROM profile_images;

-- Bad for performance: transfers all bytes even if not used
-- SELECT * FROM profile_images;
```

**Use a separate table for BLOBs to keep the main table lean:**

```sql
CREATE TABLE users (
    id    INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name  VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE
);

CREATE TABLE user_avatars (
    user_id    INT UNSIGNED PRIMARY KEY,
    image_data BLOB NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

## BLOB vs TEXT

| Feature | BLOB | TEXT |
|---|---|---|
| Content type | Binary (no charset) | Character (with charset) |
| Comparison | Byte-by-byte | Collation-aware |
| Max size | 64 KB | 64 KB |
| Use case | Images, files, encrypted data | Textual content |

## Summary

`BLOB` is the standard choice for storing binary objects up to 64 KB directly in MySQL. It is commonly used for small images, certificates, and encrypted payloads. Avoid selecting BLOB columns unless the data is actively needed, keep BLOBs in separate tables from frequently queried rows, and consider external object storage for images larger than a few kilobytes.
