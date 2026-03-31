# How to Use SHOW CREATE DATABASE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Database, Schema

Description: Learn how to use SHOW CREATE DATABASE in MySQL to view database definitions including character set and collation settings used at creation time.

---

## What Is SHOW CREATE DATABASE

`SHOW CREATE DATABASE` returns the `CREATE DATABASE` statement that would recreate a given database, including its character set and collation settings. It is useful for documentation, auditing, and reproducing a database's configuration on another server.

```sql
SHOW CREATE DATABASE database_name;
```

`SHOW CREATE SCHEMA` is a synonym and works identically:

```sql
SHOW CREATE SCHEMA database_name;
```

## Basic Usage

```sql
SHOW CREATE DATABASE myapp_production;
```

```text
+--------------------+-----------------------------------------------------------------------+
| Database           | Create Database                                                       |
+--------------------+-----------------------------------------------------------------------+
| myapp_production   | CREATE DATABASE `myapp_production` /*!40100 DEFAULT CHARACTER SET     |
|                    | utf8mb4 COLLATE utf8mb4_0900_ai_ci */ /*!80016 DEFAULT ENCRYPTION='N'*/ |
+--------------------+-----------------------------------------------------------------------+
```

The output contains versioned conditional comments (`/*!NNNNN ... */`) which MySQL uses to handle compatibility across versions.

## Understanding the Output

The output includes:

- **Character set**: The default character encoding for new tables in the database (e.g., `utf8mb4`)
- **Collation**: The default sorting and comparison rule (e.g., `utf8mb4_0900_ai_ci`)
- **Encryption**: Whether the database uses tablespace encryption (MySQL 8.0+)

## Checking Current Database Settings

You can also query the information schema directly for more parseable output:

```sql
SELECT SCHEMA_NAME, DEFAULT_CHARACTER_SET_NAME, DEFAULT_COLLATION_NAME
FROM information_schema.SCHEMATA
WHERE SCHEMA_NAME = 'myapp_production';
```

```text
+------------------+----------------------------+-----------------------+
| SCHEMA_NAME      | DEFAULT_CHARACTER_SET_NAME | DEFAULT_COLLATION_NAME|
+------------------+----------------------------+-----------------------+
| myapp_production | utf8mb4                    | utf8mb4_0900_ai_ci    |
+------------------+----------------------------+-----------------------+
```

## Using Output for Documentation

`SHOW CREATE DATABASE` output can be captured and used in runbooks or deployment scripts:

```bash
mysql -u root -p -e "SHOW CREATE DATABASE myapp_production\G" | grep "Create Database"
```

```text
Create Database: CREATE DATABASE `myapp_production` /*!40100 DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci */
```

## Recreating a Database on Another Server

The output serves as the exact statement needed to recreate the database:

```sql
-- On the source server
SHOW CREATE DATABASE source_db;

-- Copy the output and run on the target server
CREATE DATABASE `source_db`
  DEFAULT CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci
  DEFAULT ENCRYPTION='N';
```

## Comparing Character Sets Across Databases

To audit all databases for character set consistency:

```sql
SELECT SCHEMA_NAME, DEFAULT_CHARACTER_SET_NAME, DEFAULT_COLLATION_NAME
FROM information_schema.SCHEMATA
WHERE SCHEMA_NAME NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
ORDER BY SCHEMA_NAME;
```

This is useful before a migration to ensure all databases use the same encoding.

## Changing Database Character Set

If `SHOW CREATE DATABASE` reveals an outdated character set (like `latin1`), you can alter it:

```sql
ALTER DATABASE myapp_production
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

-- Verify the change
SHOW CREATE DATABASE myapp_production;
```

Note that changing the database character set does not convert existing tables - they retain their own character set settings.

## Summary

`SHOW CREATE DATABASE` displays the definition of a MySQL database including its character set, collation, and encryption settings. Use it to document database configurations, reproduce databases on other servers, and identify character set inconsistencies before migrations. For programmatic access, query `information_schema.SCHEMATA` for cleaner output.
