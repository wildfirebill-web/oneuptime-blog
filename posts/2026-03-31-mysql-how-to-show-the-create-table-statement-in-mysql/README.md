# How to Show the CREATE TABLE Statement in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, DDL, Schema Inspection, Database Management, SQL

Description: Learn how to use SHOW CREATE TABLE in MySQL to retrieve the exact DDL statement used to create a table, including indexes and constraints.

---

## Overview

`SHOW CREATE TABLE` is one of the most useful MySQL commands for schema inspection. It returns the exact `CREATE TABLE` statement that would recreate the table, including all column definitions, data types, indexes, foreign keys, character sets, and storage engine settings.

This is invaluable for documentation, migration scripts, debugging schema issues, and understanding how a table was originally defined.

## Basic Syntax

```sql
SHOW CREATE TABLE table_name;
```

## Simple Example

```sql
SHOW CREATE TABLE users;
```

```text
+-------+----------------------------------------------------------+
| Table | Create Table                                             |
+-------+----------------------------------------------------------+
| users | CREATE TABLE `users` (                                   |
|       |   `id` int NOT NULL AUTO_INCREMENT,                      |
|       |   `username` varchar(100) NOT NULL,                      |
|       |   `email` varchar(255) NOT NULL,                         |
|       |   `created_at` timestamp DEFAULT CURRENT_TIMESTAMP,      |
|       |   PRIMARY KEY (`id`),                                    |
|       |   UNIQUE KEY `email` (`email`)                           |
|       | ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4                  |
+-------+----------------------------------------------------------+
```

## Using \G for Readable Output

The default tabular output can be hard to read for complex tables. Use `\G` to display each column vertically:

```sql
SHOW CREATE TABLE orders\G
```

```text
*************************** 1. row ***************************
       Table: orders
Create Table: CREATE TABLE `orders` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `user_id` int NOT NULL,
  `total` decimal(10,2) NOT NULL,
  `status` enum('pending','paid','shipped','cancelled') NOT NULL DEFAULT 'pending',
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `idx_user_id` (`user_id`),
  CONSTRAINT `fk_orders_user` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
```

## Inspecting a Table in Another Database

You can inspect a table in a different database without switching to it:

```sql
SHOW CREATE TABLE myapp.products;
```

## Extracting the Statement in a Shell Script

To capture the CREATE TABLE output in a shell script for documentation or migration:

```bash
mysql -u root -p"${DB_PASSWORD}" "${DB_NAME}" \
  -e "SHOW CREATE TABLE users\G" 2>/dev/null | grep -A 1000 "Create Table:"
```

Or using `mysqldump` to get just the schema for one table:

```bash
mysqldump --no-data -u root -p myapp users > users_schema.sql
```

## Comparing the Output to information_schema

`SHOW CREATE TABLE` gives you the full DDL, while `information_schema.COLUMNS` gives you individual column metadata. Use both together for different needs:

```sql
-- Get column-level details
SELECT COLUMN_NAME, COLUMN_TYPE, IS_NULLABLE, COLUMN_DEFAULT
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'myapp' AND TABLE_NAME = 'users'
ORDER BY ORDINAL_POSITION;
```

```sql
-- Get the full reconstructed DDL
SHOW CREATE TABLE myapp.users;
```

## Viewing Indexes in the CREATE TABLE Output

`SHOW CREATE TABLE` includes all index definitions:

```text
CREATE TABLE `products` (
  `id` int NOT NULL AUTO_INCREMENT,
  `sku` varchar(50) NOT NULL,
  `name` varchar(200) NOT NULL,
  `price` decimal(10,2) NOT NULL,
  `category_id` int DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uq_sku` (`sku`),
  KEY `idx_category` (`category_id`),
  FULLTEXT KEY `ft_name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
```

This shows you all index types in one place - primary, unique, regular, and fulltext.

## Using SHOW CREATE TABLE for Migrations

When migrating a table to a new server or database, `SHOW CREATE TABLE` gives you the exact statement to run. You can copy it directly and execute it on the target:

```sql
-- On source server
SHOW CREATE TABLE orders\G

-- Copy the output, then on target server:
CREATE TABLE `orders` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `user_id` int NOT NULL,
  ...
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Summary

`SHOW CREATE TABLE` is the fastest way to inspect a table's complete DDL in MySQL. It shows column definitions, indexes, constraints, charset, collation, and storage engine in one command. Use the `\G` modifier for readability on complex tables, and use `mysqldump --no-data` when you need clean SQL output for migration scripts.
