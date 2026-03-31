# What Is a MySQL Schema

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Schema, Database

Description: Understand what a MySQL schema is, how it relates to a database, how to design schemas effectively, and how information_schema provides metadata access.

---

In MySQL, a schema is a synonym for a database. Unlike PostgreSQL or SQL Server - where a schema is a sub-namespace within a database - MySQL uses the terms interchangeably. Understanding this distinction avoids confusion when working across different database systems.

## Schema as Database Namespace

When you create a schema in MySQL, you create a database. The two commands are identical.

```sql
-- These are exactly equivalent in MySQL
CREATE SCHEMA myapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE myapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Both appear in SHOW DATABASES
SHOW DATABASES;
-- myapp
-- information_schema
-- mysql
-- performance_schema
-- sys
```

## The information_schema

MySQL includes a built-in read-only schema called `information_schema`. It provides metadata about all databases, tables, columns, indexes, and constraints on the server.

```sql
-- List all tables in a database
SELECT TABLE_NAME, ENGINE, TABLE_ROWS, CREATE_TIME
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'myapp'
ORDER BY TABLE_NAME;

-- List all columns in a table
SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE, COLUMN_DEFAULT, COLUMN_KEY
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'myapp'
  AND TABLE_NAME = 'users'
ORDER BY ORDINAL_POSITION;
```

## Viewing Schema Details

```sql
-- View a database's character set and collation
SHOW CREATE DATABASE myapp;

-- Check all tables and their engines in a schema
SELECT TABLE_NAME, ENGINE, TABLE_COLLATION
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'myapp';

-- Find tables not using InnoDB (potential legacy tables)
SELECT TABLE_NAME, ENGINE
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'myapp'
  AND ENGINE != 'InnoDB';
```

## Designing a Schema

Good schema design reduces storage, improves query performance, and enforces data integrity.

```sql
-- Well-designed schema example
CREATE SCHEMA ecommerce CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE ecommerce;

CREATE TABLE customers (
  id         INT AUTO_INCREMENT PRIMARY KEY,
  email      VARCHAR(255) NOT NULL,
  UNIQUE KEY uk_email (email)
) ENGINE=InnoDB;

CREATE TABLE products (
  id          INT AUTO_INCREMENT PRIMARY KEY,
  sku         VARCHAR(50) NOT NULL,
  name        VARCHAR(255) NOT NULL,
  price       DECIMAL(10,2) NOT NULL,
  UNIQUE KEY uk_sku (sku)
) ENGINE=InnoDB;

CREATE TABLE orders (
  id           INT AUTO_INCREMENT PRIMARY KEY,
  customer_id  INT NOT NULL,
  created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (customer_id) REFERENCES customers(id)
) ENGINE=InnoDB;
```

## Querying Across Schemas

MySQL allows cross-schema queries by fully qualifying table names.

```sql
-- Query across two schemas
SELECT u.email, p.amount
FROM billing.payments p
JOIN app.users u ON u.id = p.user_id
WHERE p.status = 'completed';
```

This is useful for microservice databases that share a MySQL instance.

## Schema-Level Permissions

Access to schemas is controlled through MySQL's privilege system.

```sql
-- Grant SELECT on all tables in a schema
GRANT SELECT ON myapp.* TO 'reporting_user'@'%';

-- Grant full access to one schema
GRANT ALL PRIVILEGES ON myapp.* TO 'app_user'@'%';

-- Restrict to specific tables
GRANT SELECT, INSERT ON myapp.orders TO 'api_user'@'%';

FLUSH PRIVILEGES;
```

## Schema Migrations

Schema changes are tracked using migration tools. Most ORMs and frameworks provide migration support.

```bash
# Flyway example
flyway -url=jdbc:mysql://localhost/myapp -user=root -password=pass migrate
```

```sql
-- Flyway migration file: V1__create_users.sql
CREATE TABLE users (
  id    INT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE
) ENGINE=InnoDB;
```

## Summary

In MySQL, schema and database are the same concept - a namespace for grouping related tables and objects. The `information_schema` database provides metadata about all objects across all schemas on the server. Design schemas around normalized data models with InnoDB, utf8mb4, and clear foreign key relationships. Use migration tools like Flyway or Liquibase to manage schema evolution in production.
