# How to Use CREATE SCHEMA Statement in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Schema, Database, DDL, Character Set

Description: Learn how CREATE SCHEMA works in MySQL as a synonym for CREATE DATABASE, including charset and collation options for schema configuration.

---

## Overview

In MySQL, `CREATE SCHEMA` is a synonym for `CREATE DATABASE`. The two statements are interchangeable, and both create a new database (also called a schema) that serves as a logical namespace for tables, views, stored procedures, and other objects. `CREATE SCHEMA` is preferred by developers coming from a standard SQL background where "schema" is the common term.

## Basic Syntax

```sql
CREATE SCHEMA [IF NOT EXISTS] schema_name
  [CHARACTER SET charset_name]
  [COLLATE collation_name]
  [ENCRYPTION {'Y' | 'N'}];
```

## Creating a Simple Schema

```sql
CREATE SCHEMA myapp;
```

This creates a new database named `myapp` with the server's default character set and collation.

## Using IF NOT EXISTS

Prevent errors when the schema might already exist:

```sql
CREATE SCHEMA IF NOT EXISTS myapp;
```

## Specifying Character Set and Collation

For applications that require a specific character encoding, set the character set and collation at creation time:

```sql
CREATE SCHEMA myapp
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

Using `utf8mb4` is recommended for full Unicode support, including emoji and supplementary characters.

## Encryption (MySQL 8.0+)

MySQL 8.0 supports schema-level encryption when the keyring plugin is configured:

```sql
CREATE SCHEMA secure_data
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci
  ENCRYPTION 'Y';
```

When encryption is enabled at the schema level, all InnoDB tables created in that schema are encrypted by default.

## Verifying the Schema Was Created

```sql
SHOW SCHEMAS;
-- or
SHOW DATABASES;
```

Both commands show the same list. To see the schema definition:

```sql
SHOW CREATE SCHEMA myapp;
-- Equivalent to:
SHOW CREATE DATABASE myapp;
```

## Selecting the Schema

Switch to the schema to create objects inside it:

```sql
USE myapp;

CREATE TABLE users (
  id   INT NOT NULL AUTO_INCREMENT,
  name VARCHAR(100) NOT NULL,
  PRIMARY KEY (id)
);
```

## Required Privileges

Creating a schema requires the `CREATE` privilege:

```sql
GRANT CREATE ON *.* TO 'dev_user'@'localhost';
-- Or for a specific schema:
GRANT ALL ON myapp.* TO 'dev_user'@'localhost';
```

## Altering a Schema After Creation

You can change the default character set or collation after creation:

```sql
ALTER SCHEMA myapp
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;
```

Note: This changes the default for new tables only; existing table character sets are not altered automatically.

## Summary

`CREATE SCHEMA` in MySQL is identical to `CREATE DATABASE` and creates a new logical namespace for database objects. Use it with `CHARACTER SET` and `COLLATE` options to establish the default encoding for all tables in the schema, and optionally enable encryption in MySQL 8.0. Adopt `CREATE SCHEMA` when following SQL standard conventions or working in environments that use schema as the primary organizational unit.
