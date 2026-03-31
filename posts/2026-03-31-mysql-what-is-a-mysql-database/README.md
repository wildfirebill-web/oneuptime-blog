# What Is a MySQL Database

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Database, Schema

Description: Understand what a MySQL database is, how it differs from a schema, how to create and manage databases, and best practices for database organization.

---

In MySQL, a "database" is a named namespace that groups related tables, views, stored procedures, functions, and other objects. A single MySQL server instance can host many databases, each isolated from the others.

## Database vs Schema

In MySQL, the terms "database" and "schema" are synonymous and interchangeable. Running `CREATE DATABASE myapp` and `CREATE SCHEMA myapp` produce identical results. This differs from PostgreSQL and SQL Server, where schemas are sub-namespaces within a database.

```sql
-- These two statements are equivalent in MySQL
CREATE DATABASE myapp;
CREATE SCHEMA myapp;

-- Verify
SHOW DATABASES;
-- information_schema
-- myapp
-- mysql
-- performance_schema
-- sys
```

## Creating a Database

```sql
-- Basic creation
CREATE DATABASE myapp;

-- Specify character set and collation (recommended)
CREATE DATABASE myapp
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

-- IF NOT EXISTS prevents an error if it already exists
CREATE DATABASE IF NOT EXISTS myapp
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

Always use `utf8mb4` (not the misleadingly-named `utf8`, which is actually UTF-8 limited to 3 bytes) to support the full Unicode character set including emoji.

## Selecting a Database

Once a database is created, you must select it before running table-level statements.

```sql
-- Select active database
USE myapp;

-- Check current database
SELECT DATABASE();

-- Alternatively, qualify all references with the database name
SELECT * FROM myapp.users;
```

## Viewing Database Information

```sql
-- List all databases
SHOW DATABASES;

-- View creation statement of a database
SHOW CREATE DATABASE myapp;
-- CREATE DATABASE `myapp` /*!40100 DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci */

-- Show tables in the current database
SHOW TABLES;

-- Show tables in a specific database without switching
SHOW TABLES FROM myapp;
```

## System Databases

MySQL ships with several built-in system databases:

- `mysql` - stores user accounts, privileges, and server configuration
- `information_schema` - read-only views of metadata (tables, columns, indexes)
- `performance_schema` - runtime performance metrics and event instrumentation
- `sys` - human-readable views over performance_schema data

```sql
-- Query information_schema for table details
SELECT TABLE_NAME, TABLE_ROWS, DATA_LENGTH
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'myapp'
ORDER BY DATA_LENGTH DESC;
```

## Dropping a Database

```sql
-- Drop a database and everything in it (irreversible)
DROP DATABASE myapp;

-- Safe version
DROP DATABASE IF EXISTS myapp;
```

`DROP DATABASE` deletes all tables, views, stored procedures, and data files for that database. There is no built-in recycle bin.

## Renaming a Database

MySQL has no `RENAME DATABASE` command. The standard approach is to create a new database and move tables.

```bash
# Rename by dumping and reimporting
mysqldump -u root -p myapp > myapp_backup.sql
mysql -u root -p -e "CREATE DATABASE newapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci"
mysql -u root -p newapp < myapp_backup.sql
mysql -u root -p -e "DROP DATABASE myapp"
```

## Summary

A MySQL database is a named namespace containing all related objects for an application or service. Create databases with `CREATE DATABASE`, always specifying `utf8mb4` character set for full Unicode support. Use `information_schema` to query metadata about your databases and tables. A single MySQL server instance can host hundreds of databases, each with independent schemas and access controls.
