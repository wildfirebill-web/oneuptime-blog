# How to Configure the MySQL Character Set to utf8mb4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, utf8mb4, Character Set, Collation, Internationalization

Description: Configure MySQL to use utf8mb4 as the default character set to properly store all Unicode characters including emojis and multilingual text.

---

## Why utf8mb4 Instead of utf8

MySQL's `utf8` character set is a 3-byte encoding that does not support characters outside the Basic Multilingual Plane - this excludes emojis, some Chinese characters, and rare Unicode symbols. The `utf8mb4` character set is the true 4-byte UTF-8 encoding and supports the full Unicode range.

Always use `utf8mb4` for new databases. The older `utf8` alias is misleading and should be avoided.

## Setting utf8mb4 in my.cnf

Open the MySQL configuration file and add the following settings under `[mysqld]`, `[client]`, and `[mysql]`:

```text
[mysqld]
character-set-server    = utf8mb4
collation-server        = utf8mb4_unicode_ci
init_connect            = 'SET NAMES utf8mb4'

[client]
default-character-set   = utf8mb4

[mysql]
default-character-set   = utf8mb4
```

Save the file and restart MySQL:

```bash
sudo systemctl restart mysql
```

## Verifying the Server Character Set

After restarting, check that the server is using utf8mb4:

```sql
SHOW VARIABLES LIKE 'character%';
SHOW VARIABLES LIKE 'collation%';
```

Expected output:

```text
+--------------------------+--------------------+
| Variable_name            | Value              |
+--------------------------+--------------------+
| character_set_client     | utf8mb4            |
| character_set_connection | utf8mb4            |
| character_set_database   | utf8mb4            |
| character_set_results    | utf8mb4            |
| character_set_server     | utf8mb4            |
| character_set_system     | utf8mb3            |
+--------------------------+--------------------+
```

Note: `character_set_system` is always `utf8mb3` internally and cannot be changed.

## Creating a Database with utf8mb4

When creating a new database, explicitly specify the character set:

```sql
CREATE DATABASE myapp
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

## Creating Tables with utf8mb4

Specify the character set at the table level:

```sql
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  username VARCHAR(50) NOT NULL,
  bio TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_unicode_ci;
```

## Choosing a Collation

The collation controls how strings are sorted and compared. Common options for utf8mb4:

- `utf8mb4_unicode_ci` - Unicode standard sorting, case-insensitive (recommended)
- `utf8mb4_general_ci` - Faster but less accurate Unicode sorting
- `utf8mb4_bin` - Binary comparison, case-sensitive
- `utf8mb4_0900_ai_ci` - Accent-insensitive, case-insensitive (MySQL 8.0 default)

For most applications, `utf8mb4_unicode_ci` or `utf8mb4_0900_ai_ci` (MySQL 8.0+) is a good choice.

## Converting an Existing Database to utf8mb4

To convert an existing database from `utf8` to `utf8mb4`:

```sql
-- Convert database
ALTER DATABASE myapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Convert each table
ALTER TABLE users CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
ALTER TABLE posts CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

For large databases, use a script to convert all tables at once:

```sql
SELECT CONCAT(
  'ALTER TABLE `', TABLE_NAME,
  '` CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;'
)
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'myapp'
  AND TABLE_TYPE = 'BASE TABLE';
```

Run the generated statements to convert all tables.

## Application Connection String

Ensure your application connects using utf8mb4. For example, in Node.js with mysql2:

```javascript
const connection = mysql.createConnection({
  host: 'localhost',
  user: 'root',
  password: 'password',
  database: 'myapp',
  charset: 'utf8mb4'
});
```

In Python with mysql-connector:

```python
import mysql.connector

conn = mysql.connector.connect(
    host='localhost',
    user='root',
    password='password',
    database='myapp',
    charset='utf8mb4',
    collation='utf8mb4_unicode_ci'
)
```

## Testing Emoji Storage

Verify that emojis can be stored and retrieved correctly:

```sql
INSERT INTO users (username, bio) VALUES ('testuser', 'I love pizza!');

SELECT username, bio FROM users WHERE username = 'testuser';
```

If the emoji stores as `????`, the connection or table is still using the 3-byte `utf8` charset.

## Summary

Configure MySQL to use `utf8mb4` by setting `character-set-server = utf8mb4` and `collation-server = utf8mb4_unicode_ci` in `my.cnf`, then restart the server. Always create databases and tables with explicit utf8mb4 declarations. For existing databases, use `ALTER TABLE ... CONVERT TO CHARACTER SET utf8mb4` to migrate. The `utf8mb4` charset is required for storing the full Unicode range including emojis and multilingual content.
