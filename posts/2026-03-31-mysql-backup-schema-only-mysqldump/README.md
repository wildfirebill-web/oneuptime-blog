# How to Back Up Only the Schema (No Data) with mysqldump in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Backup, mysqldump, Schema, Migration

Description: Learn how to export only the database schema (DDL) without data using mysqldump --no-data for migrations, documentation, and version control.

---

Exporting only the schema (table definitions, indexes, stored procedures, views, and triggers) without row data is useful in many scenarios: setting up a development environment from production schema, generating DDL documentation, storing schema in version control, or creating a target database before migrating data separately. The `mysqldump --no-data` flag handles this cleanly.

## Basic Schema-Only Export

```bash
mysqldump -u root -p --no-data database_name > schema_only.sql
```

## Schema for a Specific Database

```bash
mysqldump -u root -p --no-data myapp > /backup/myapp_schema_$(date +%Y%m%d).sql
```

## Schema for a Specific Table

```bash
mysqldump -u root -p --no-data myapp orders > /backup/orders_schema.sql
```

## Including Stored Procedures, Functions, and Events

By default, schema exports include triggers. Add these flags for complete DDL:

```bash
mysqldump -u root -p \
  --no-data \
  --routines \
  --events \
  --triggers \
  myapp \
  > /backup/myapp_full_schema.sql
```

## Schema for All Databases

```bash
mysqldump -u root -p \
  --all-databases \
  --no-data \
  --routines \
  --events \
  > /backup/all_schemas_$(date +%Y%m%d).sql
```

## Exporting Schema with DROP TABLE Statements

For clean re-creation, include `DROP TABLE IF EXISTS` before each `CREATE TABLE`:

```bash
mysqldump -u root -p \
  --no-data \
  --add-drop-table \
  myapp \
  > /backup/myapp_schema_drop_first.sql
```

## Creating a Development Environment Schema

```bash
# Export production schema
mysqldump -u root -p \
  --no-data \
  --routines \
  --events \
  prod_myapp \
  > /tmp/dev_schema.sql

# Apply to development database
mysql -u root -p dev_myapp < /tmp/dev_schema.sql
```

## Viewing the Schema Export

Check what the schema dump contains:

```bash
# Preview the output
mysqldump -u root -p --no-data myapp | head -100
```

A typical schema export looks like:

```sql
-- MySQL dump 10.13  Distrib 8.0.36

CREATE DATABASE /*!32312 IF NOT EXISTS*/ `myapp`;
USE `myapp`;

DROP TABLE IF EXISTS `orders`;
CREATE TABLE `orders` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `user_id` int NOT NULL,
  `total` decimal(12,2) NOT NULL,
  `status` enum('pending','paid','shipped') NOT NULL,
  `created_at` timestamp DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Storing Schema in Git

```bash
# Export schema and commit to version control
mysqldump -u root -p \
  --no-data \
  --skip-comments \
  --compact \
  myapp \
  > /repo/db/schema.sql

cd /repo
git add db/schema.sql
git commit -m "Update database schema - $(date +%Y-%m-%d)"
```

The `--compact` flag removes MySQL version comments and header/footer lines for cleaner diffs. The `--skip-comments` flag removes timestamp comments that would otherwise generate noise in git diffs.

## Comparing Two Schema Versions

```bash
diff /backup/myapp_schema_yesterday.sql /backup/myapp_schema_today.sql
```

## Summary

`mysqldump --no-data` exports only DDL statements - `CREATE TABLE`, `CREATE INDEX`, `CREATE PROCEDURE`, etc. - without any row data. Always add `--routines --events --triggers` for a complete schema export. Use `--compact --skip-comments` when storing in version control to minimize diff noise, and `--add-drop-table` when the schema will be applied to an existing database.
