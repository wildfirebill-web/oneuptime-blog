# How to Use CREATE DATABASE IF NOT EXISTS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Database Management, DDL, Schema Design, SQL

Description: Learn how to use CREATE DATABASE IF NOT EXISTS in MySQL to safely create databases without errors when they already exist.

---

## Overview

When writing scripts or application setup code, you often need to create a database only if it does not already exist. MySQL provides the `CREATE DATABASE IF NOT EXISTS` syntax to handle this scenario gracefully - instead of throwing an error when the database exists, it silently skips the creation and continues execution.

This is particularly useful in deployment scripts, migrations, and Docker entrypoint scripts where idempotent database setup is essential.

## Basic Syntax

```sql
CREATE DATABASE IF NOT EXISTS database_name;
```

If `database_name` already exists, MySQL issues a Note-level warning but does not raise an error. If it does not exist, the database is created.

## Creating a Database with Character Set and Collation

Always specify the character set and collation when creating a database to avoid encoding issues:

```sql
CREATE DATABASE IF NOT EXISTS myapp
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

Using `utf8mb4` supports the full Unicode range including emoji and multi-byte characters that the older `utf8` alias does not handle correctly.

## Checking Whether the Database Was Created

After running the command, verify whether the database exists:

```sql
SHOW DATABASES LIKE 'myapp';
```

Expected output:

```text
+------------------+
| Database (myapp) |
+------------------+
| myapp            |
+------------------+
```

## Using in a Shell Script

A common use case is running this in a deployment shell script:

```bash
mysql -u root -p"${MYSQL_ROOT_PASSWORD}" \
  -e "CREATE DATABASE IF NOT EXISTS ${DB_NAME} CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

This command is safe to run multiple times without failing.

## Using in Application Initialization

When building Node.js applications, you can run this on startup:

```javascript
const mysql = require('mysql2/promise');

async function initDatabase() {
  const conn = await mysql.createConnection({
    host: 'localhost',
    user: 'root',
    password: process.env.DB_PASSWORD
  });

  await conn.query(
    `CREATE DATABASE IF NOT EXISTS \`${process.env.DB_NAME}\`
     CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci`
  );

  await conn.end();
}
```

## Creating Multiple Databases

You can run multiple `CREATE DATABASE IF NOT EXISTS` statements in sequence:

```sql
CREATE DATABASE IF NOT EXISTS app_development
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE DATABASE IF NOT EXISTS app_test
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE DATABASE IF NOT EXISTS app_staging
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

## Viewing Warning Details

If the database already exists, MySQL issues a warning. You can inspect it:

```sql
CREATE DATABASE IF NOT EXISTS myapp;
SHOW WARNINGS;
```

```text
+-------+------+------------------------------------------------+
| Level | Code | Message                                        |
+-------+------+------------------------------------------------+
| Note  | 1007 | Can't create database 'myapp'; database exists |
+-------+------+------------------------------------------------+
```

The warning level is `Note` (not `Warning` or `Error`), so it does not interrupt script execution.

## Differences Between IF NOT EXISTS and Without It

```sql
-- This will throw ERROR 1007 if the database already exists:
CREATE DATABASE myapp;

-- This will silently succeed even if the database already exists:
CREATE DATABASE IF NOT EXISTS myapp;
```

Always prefer the `IF NOT EXISTS` form in scripts to ensure idempotency.

## Granting Privileges After Creation

After creating the database, grant privileges to an application user:

```sql
CREATE DATABASE IF NOT EXISTS myapp
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

GRANT ALL PRIVILEGES ON myapp.* TO 'appuser'@'%';
FLUSH PRIVILEGES;
```

## Summary

`CREATE DATABASE IF NOT EXISTS` is the recommended way to create MySQL databases in scripts, migrations, and application initialization code. It prevents errors when the database already exists, making your scripts idempotent. Always pair it with explicit `CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci` to ensure correct Unicode support. Combined with proper user grants, it gives you a complete, repeatable database setup routine.
