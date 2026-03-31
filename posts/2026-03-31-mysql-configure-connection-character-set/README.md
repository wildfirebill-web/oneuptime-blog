# How to Configure Connection Character Set in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Character Set, Connection, Configuration, Client

Description: Learn how to configure the character set for MySQL client connections to avoid encoding mismatches between your application and the database.

---

## Why Connection Character Set Matters

MySQL tracks character encoding at multiple levels: server, database, table, column, and connection. If the connection character set does not match the server or column character set, MySQL may silently mis-encode data or raise errors. For example, if your application sends `utf8mb4` data over a `latin1` connection, emoji and multibyte characters will be stored as garbled bytes.

## The Three Key Connection Variables

MySQL uses three session variables to handle client-server encoding:

```sql
SHOW VARIABLES LIKE 'character_set_client';
SHOW VARIABLES LIKE 'character_set_connection';
SHOW VARIABLES LIKE 'character_set_results';
```

- **`character_set_client`** - Encoding the client sends data in.
- **`character_set_connection`** - Encoding used internally for literal strings.
- **`character_set_results`** - Encoding MySQL uses when returning results to the client.

All three should be set to the same value, typically `utf8mb4`.

## Method 1 - SET NAMES

The simplest way to configure all three at once is `SET NAMES`:

```sql
SET NAMES 'utf8mb4';
```

This is equivalent to:

```sql
SET character_set_client     = utf8mb4;
SET character_set_connection = utf8mb4;
SET character_set_results    = utf8mb4;
```

Run this immediately after opening each connection.

## Method 2 - SET NAMES with Collation

Pair `SET NAMES` with an explicit collation for full control:

```sql
SET NAMES 'utf8mb4' COLLATE 'utf8mb4_unicode_ci';
```

## Method 3 - Configure in my.cnf

Set the default at the server and client level via `my.cnf` so all connections default to `utf8mb4`:

```ini
[mysqld]
character_set_server   = utf8mb4
collation_server       = utf8mb4_unicode_ci

[client]
default-character-set  = utf8mb4

[mysql]
default-character-set  = utf8mb4
```

## Method 4 - Connection String Options

Most drivers let you specify the character set in the connection string, which avoids the need for a separate `SET NAMES` call:

```text
# Python (mysql-connector-python)
charset=utf8mb4

# Node.js (mysql2)
charset: 'utf8mb4'

# PHP (PDO DSN)
charset=utf8mb4

# Java (JDBC URL)
?characterEncoding=UTF-8&useUnicode=true
```

## Verifying the Active Connection Encoding

```sql
SHOW SESSION VARIABLES LIKE 'character_set%';
SHOW SESSION VARIABLES LIKE 'collation%';
```

Expected output for a correctly configured `utf8mb4` connection:

```text
character_set_client     | utf8mb4
character_set_connection | utf8mb4
character_set_results    | utf8mb4
collation_connection     | utf8mb4_unicode_ci
```

## Summary

Always set the connection character set to `utf8mb4` to match modern MySQL schemas. Use `SET NAMES 'utf8mb4'` as a post-connect command, configure `my.cnf` for server-wide defaults, or pass `charset=utf8mb4` in your driver's connection options. Verify the active settings with `SHOW SESSION VARIABLES` when debugging encoding issues.
