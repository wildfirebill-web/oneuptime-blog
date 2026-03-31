# How to Fix ERROR 1366 Incorrect String Value in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Error, Charset, Troubleshooting, Database

Description: Learn how to fix MySQL ERROR 1366 Incorrect string value caused by emoji or multibyte characters when the column charset is not utf8mb4.

---

## What Is ERROR 1366?

```
ERROR 1366 (HY000): Incorrect string value: '\xF0\x9F\x98\x80' for column 'name' at row 1
```

This error appears when you try to insert a character that cannot be encoded in the column's character set. The most common cause is inserting emoji or 4-byte Unicode characters into a column that uses `utf8` instead of `utf8mb4`.

MySQL's `utf8` charset only supports up to 3-byte characters. Emoji and many rare Unicode code points require 4 bytes, which only `utf8mb4` supports.

## Identify the Affected Column's Charset

```sql
SHOW CREATE TABLE users\G
-- or
SELECT TABLE_NAME, COLUMN_NAME, CHARACTER_SET_NAME, COLLATION_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'users';
```

If the column shows `utf8` (which is actually `utf8mb3` in MySQL 8.0), that is the root cause.

## Fix 1: Convert the Column to utf8mb4

```sql
ALTER TABLE users
  MODIFY COLUMN name VARCHAR(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

## Fix 2: Convert the Entire Table

```sql
ALTER TABLE users
  CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

## Fix 3: Convert the Entire Database

```sql
ALTER DATABASE mydb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Then convert each table:

```sql
SELECT CONCAT(
  'ALTER TABLE `', TABLE_NAME, '` CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;'
)
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_TYPE = 'BASE TABLE';
```

## Fix 4: Set the Connection Charset

Sometimes the column is already `utf8mb4` but the connection is negotiating `utf8`. Add this after connecting:

```sql
SET NAMES utf8mb4;
```

Or in the connection string (Node.js example):

```javascript
const connection = mysql.createConnection({
  host: 'localhost',
  user: 'root',
  password: 'pass',
  database: 'mydb',
  charset: 'utf8mb4'
});
```

In Python:

```python
import mysql.connector
conn = mysql.connector.connect(
    host='localhost',
    user='root',
    password='pass',
    database='mydb',
    charset='utf8mb4'
)
```

## Fix 5: Update my.cnf to Default to utf8mb4

Edit `/etc/mysql/mysql.conf.d/mysqld.cnf`:

```ini
[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[client]
default-character-set = utf8mb4
```

Restart MySQL:

```bash
sudo systemctl restart mysql
```

## Verify the Fix

```sql
SHOW VARIABLES LIKE 'character_set%';
SHOW VARIABLES LIKE 'collation%';
```

Then retry the insert:

```sql
INSERT INTO users (name) VALUES ('Hello ');
```

## Summary

ERROR 1366 is almost always caused by inserting 4-byte Unicode characters (emoji, rare CJK characters) into a `utf8` column. Fix it by converting affected columns or tables to `utf8mb4`, setting `SET NAMES utf8mb4` on the connection, and updating `my.cnf` to make `utf8mb4` the server default. MySQL's `utf8` is a 3-byte subset; `utf8mb4` is the true full-Unicode charset.
