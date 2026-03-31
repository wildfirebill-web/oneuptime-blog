# How to Set the Character Set for a MySQL Database

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Character Set, Collation, Unicode, Configuration

Description: Learn how to set and change the default character set and collation for a MySQL database to ensure correct text storage and comparison behavior.

---

## Why Character Set Matters

The character set determines which characters can be stored in text columns. The collation determines how those characters are compared and sorted. Choosing the wrong character set leads to data loss, incorrect sorting, or errors when inserting text containing non-ASCII characters like accented letters, emoji, or CJK characters.

For new databases in MySQL 8.0+, `utf8mb4` with `utf8mb4_unicode_ci` or `utf8mb4_0900_ai_ci` is the recommended default.

## Setting Character Set When Creating a Database

```sql
CREATE DATABASE myapp
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

## Changing Character Set on an Existing Database

Changing the database character set only affects the default for new tables created afterward. Existing tables and columns retain their own character sets.

```sql
ALTER DATABASE myapp
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;
```

## Checking the Current Character Set

```sql
-- Check database defaults
SELECT schema_name, default_character_set_name, default_collation_name
FROM information_schema.schemata
WHERE schema_name = 'myapp';
```

```text
+-------------+----------------------------+-----------------------+
| schema_name | default_character_set_name | default_collation_name|
+-------------+----------------------------+-----------------------+
| myapp       | utf8mb4                    | utf8mb4_unicode_ci    |
+-------------+----------------------------+-----------------------+
```

## Setting the Server Default in my.cnf

To make utf8mb4 the default for all new databases and connections, configure it in `/etc/mysql/my.cnf`:

```text
[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[client]
default-character-set = utf8mb4
```

After saving, restart MySQL:

```bash
sudo systemctl restart mysql
```

## Session Character Set

For a connection, set the character set explicitly to avoid encoding mismatches between client and server:

```sql
SET NAMES 'utf8mb4';
-- Equivalent to:
SET character_set_client = utf8mb4;
SET character_set_connection = utf8mb4;
SET character_set_results = utf8mb4;
```

Most drivers set this automatically. In Python:

```python
conn = mysql.connector.connect(
    host='localhost',
    user='app',
    password='pass',
    database='myapp',
    charset='utf8mb4'
)
```

## Converting Existing Tables

When migrating an existing database to utf8mb4, convert tables one by one:

```sql
-- Convert a table and all its text columns
ALTER TABLE articles
  CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

For an entire database, generate ALTER statements for all tables:

```sql
SELECT CONCAT(
  'ALTER TABLE ', TABLE_NAME,
  ' CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;'
) AS alter_stmt
FROM information_schema.tables
WHERE TABLE_SCHEMA = 'myapp' AND TABLE_TYPE = 'BASE TABLE';
```

## Choosing a Collation

| Collation | Behavior |
|---|---|
| `utf8mb4_unicode_ci` | Unicode standard, case-insensitive |
| `utf8mb4_0900_ai_ci` | Unicode 9.0, accent-insensitive, faster |
| `utf8mb4_bin` | Binary comparison, case-sensitive |

`utf8mb4_0900_ai_ci` is the default in MySQL 8.0 and is recommended for new applications.

## Summary

Set the character set for a MySQL database with `CREATE DATABASE ... CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci`. For servers, configure `character-set-server` in `my.cnf`. For connections, use `SET NAMES 'utf8mb4'`. When migrating existing databases, use `ALTER TABLE ... CONVERT TO CHARACTER SET` to update individual tables. Always use `utf8mb4` over the legacy `utf8` alias, which only supports up to 3-byte characters.
