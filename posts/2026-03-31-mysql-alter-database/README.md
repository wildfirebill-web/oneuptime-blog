# How to Use ALTER DATABASE Statement in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Database, DDL, Character Set, Encryption

Description: Learn how to use ALTER DATABASE in MySQL to change the default character set, collation, or encryption settings of an existing database.

---

## Overview

The `ALTER DATABASE` statement modifies the properties of an existing MySQL database. You can use it to change the default character set and collation for new tables, enable or disable schema-level encryption, and update the read-only attribute. `ALTER SCHEMA` is a synonym for `ALTER DATABASE`.

## Basic Syntax

```sql
ALTER DATABASE database_name
  [CHARACTER SET charset_name]
  [COLLATE collation_name]
  [ENCRYPTION {'Y' | 'N'}]
  [READ ONLY {DEFAULT | 0 | 1}];
```

## Changing the Default Character Set and Collation

The most common use of `ALTER DATABASE` is to update the default character set when migrating from older encodings:

```sql
-- Switch from latin1 to utf8mb4
ALTER DATABASE myapp
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

This changes the default for newly created tables in the database. Existing tables are not automatically converted.

## Converting Existing Tables After ALTER DATABASE

To align existing tables with the new default, convert them individually:

```sql
ALTER TABLE users
  CONVERT TO CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

Or generate the ALTER TABLE statements dynamically:

```sql
SELECT CONCAT(
  'ALTER TABLE ', TABLE_NAME,
  ' CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;'
) AS stmt
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'myapp'
  AND TABLE_TYPE = 'BASE TABLE';
```

## Enabling Encryption (MySQL 8.0+)

Turn on schema-level encryption when the keyring plugin is active:

```sql
ALTER DATABASE secure_data
  ENCRYPTION = 'Y';
```

After this, new tables created in `secure_data` will be encrypted by default. Existing tables remain unencrypted until you explicitly encrypt them:

```sql
ALTER TABLE sensitive_records ENCRYPTION = 'Y';
```

## Making a Database Read-Only (MySQL 8.0.22+)

The `READ ONLY` option prevents any DDL or DML changes to the database, useful for database migrations or archiving:

```sql
-- Lock the database for reading only
ALTER DATABASE archive_2023 READ ONLY = 1;

-- Restore write access
ALTER DATABASE archive_2023 READ ONLY = 0;
```

## Viewing Current Database Properties

```sql
SHOW CREATE DATABASE myapp;
```

Or query `INFORMATION_SCHEMA`:

```sql
SELECT DEFAULT_CHARACTER_SET_NAME, DEFAULT_COLLATION_NAME
FROM INFORMATION_SCHEMA.SCHEMATA
WHERE SCHEMA_NAME = 'myapp';
```

## Required Privileges

`ALTER DATABASE` requires the `ALTER` privilege on the database:

```sql
GRANT ALTER ON myapp.* TO 'dba_user'@'localhost';
```

## Typical Migration Workflow

```sql
-- 1. Check current settings
SHOW CREATE DATABASE myapp;

-- 2. Update database defaults
ALTER DATABASE myapp
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;

-- 3. Convert existing tables (run for each table)
ALTER TABLE orders CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
ALTER TABLE users  CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;

-- 4. Verify
SHOW CREATE DATABASE myapp;
```

## Summary

`ALTER DATABASE` changes the default character set, collation, encryption, or read-only status of an existing MySQL database. It is commonly used during charset migrations, security hardening, and database lifecycle management. Remember that changing the database default only affects new tables - existing tables must be converted separately to match the new settings.
