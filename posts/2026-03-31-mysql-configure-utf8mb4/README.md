# How to Configure MySQL Character Set to utf8mb4

Author: [OneUptime](https://oneuptime.com)

Tags: MySQL, Configuration, Character Set, utf8mb4, Internationalization

Description: Configure MySQL to use utf8mb4 as the default character set to support full Unicode including emoji, by updating my.cnf and converting existing databases.

---

## How It Works

MySQL's legacy `utf8` character set only encodes the Basic Multilingual Plane (BMP) using up to 3 bytes per character. `utf8mb4` is the proper 4-byte UTF-8 encoding that supports the full Unicode range, including emoji (U+1F600 and above), Supplementary Multilingual Plane characters, and rare CJK extensions. Modern MySQL deployments should always use `utf8mb4`.

```mermaid
flowchart LR
    A[Set character-set-server = utf8mb4 in my.cnf] --> B[Restart MySQL]
    B --> C[New databases use utf8mb4 by default]
    C --> D[Convert existing databases]
    D --> E[Verify with SHOW VARIABLES]
```

## Checking the Current Character Set

```sql
SHOW VARIABLES LIKE 'character_set%';
SHOW VARIABLES LIKE 'collation%';
```

```text
+--------------------------+-------------------+
| Variable_name            | Value             |
+--------------------------+-------------------+
| character_set_client     | utf8mb3           |
| character_set_connection | utf8mb3           |
| character_set_database   | utf8mb3           |
| character_set_results    | utf8mb3           |
| character_set_server     | utf8mb3           |
| character_set_system     | utf8mb3           |
+--------------------------+-------------------+
```

## Step 1 - Update my.cnf

Edit the MySQL configuration file.

```bash
# Ubuntu / Debian
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf

# RHEL / CentOS / Rocky / AlmaLinux
sudo nano /etc/my.cnf
```

Add or update the following sections.

```ini
[mysqld]
character-set-server  = utf8mb4
collation-server      = utf8mb4_unicode_ci
skip-character-set-client-handshake

[client]
default-character-set = utf8mb4

[mysql]
default-character-set = utf8mb4
```

The `skip-character-set-client-handshake` directive forces the server to use its own character set regardless of what the client requests. Omit it if your clients need to negotiate different character sets.

## Step 2 - Restart MySQL

```bash
# Ubuntu / Debian
sudo systemctl restart mysql

# RHEL-based
sudo systemctl restart mysqld
```

## Step 3 - Verify the New Settings

```sql
SHOW VARIABLES LIKE 'character_set%';
SHOW VARIABLES LIKE 'collation%';
```

```text
+--------------------------+--------------------+
| Variable_name            | Value              |
+--------------------------+--------------------+
| character_set_server     | utf8mb4            |
| character_set_database   | utf8mb4            |
| character_set_client     | utf8mb4            |
| character_set_connection | utf8mb4            |
| character_set_results    | utf8mb4            |
+--------------------------+--------------------+

+----------------------+--------------------+
| Variable_name        | Value              |
+----------------------+--------------------+
| collation_server     | utf8mb4_unicode_ci |
| collation_database   | utf8mb4_unicode_ci |
| collation_connection | utf8mb4_unicode_ci |
+----------------------+--------------------+
```

## Step 4 - Convert Existing Databases

Changing `my.cnf` only affects new databases and connections. Existing databases, tables, and columns must be converted explicitly.

### Convert a Database

```sql
ALTER DATABASE myapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### Convert a Table

```sql
ALTER TABLE users
    CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### Convert a Specific Column

```sql
ALTER TABLE posts
    MODIFY COLUMN body TEXT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### Generate ALTER Statements for All Tables in a Database

```sql
SELECT CONCAT(
    'ALTER TABLE `', TABLE_NAME, '` ',
    'CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;'
)
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'myapp'
  AND TABLE_TYPE = 'BASE TABLE';
```

Copy and run the output to convert all tables at once.

## Step 5 - Test Emoji Storage

```sql
CREATE TABLE emoji_test (
    id  INT AUTO_INCREMENT PRIMARY KEY,
    txt VARCHAR(100) CHARACTER SET utf8mb4
);

INSERT INTO emoji_test (txt) VALUES ('Hello World 😀🎉🌍');

SELECT * FROM emoji_test;
```

```text
+----+----------------------+
| id | txt                  |
+----+----------------------+
|  1 | Hello World 😀🎉🌍  |
+----+----------------------+
```

## Application-Level Considerations

Always specify `charset=utf8mb4` in your connection string.

```python
# Python (mysql-connector)
conn = mysql.connector.connect(charset='utf8mb4', ...)
```

```php
// PHP PDO
$dsn = 'mysql:host=localhost;dbname=myapp;charset=utf8mb4';
```

```javascript
// Node.js (mysql2)
const pool = mysql.createPool({ charset: 'utf8mb4', ... });
```

## InnoDB Row Format Consideration

`utf8mb4` uses up to 4 bytes per character. The InnoDB DYNAMIC row format (default in MySQL 5.7.9+) handles the maximum key prefix length correctly. If you have older tables using COMPACT format:

```sql
ALTER TABLE mytable ROW_FORMAT=DYNAMIC;
```

## utf8mb4_unicode_ci vs. utf8mb4_0900_ai_ci

MySQL 8.0 introduced `utf8mb4_0900_ai_ci` (Unicode 9.0, accent-insensitive, case-insensitive) as a more correct and faster alternative to `utf8mb4_unicode_ci`. For new databases on MySQL 8.0:

```sql
CREATE DATABASE newapp CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

For compatibility with applications that were written for MySQL 5.7, `utf8mb4_unicode_ci` is safer.

## Summary

Setting `character-set-server = utf8mb4` and `collation-server = utf8mb4_unicode_ci` in `my.cnf` ensures all new databases use the correct Unicode character set. Existing databases and tables must be converted with `ALTER DATABASE` and `ALTER TABLE ... CONVERT TO CHARACTER SET utf8mb4`. Always set `charset=utf8mb4` in application connection strings to prevent the client from falling back to a narrower encoding.
