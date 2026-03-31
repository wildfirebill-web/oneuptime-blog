# How to Use MySQL Workbench Forward Engineering

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Workbench, Forward Engineering, DDL, Schema

Description: Learn how to use MySQL Workbench Forward Engineering to generate DDL SQL from EER models and apply schema changes directly to a live MySQL server.

---

## Introduction

Forward Engineering in MySQL Workbench converts a visual data model (EER diagram) into SQL DDL statements. It bridges the gap between visual database design and actual database creation, allowing you to generate `CREATE TABLE` scripts, apply them to a live server, or save them for version control. This is the workflow for model-driven database development.

## Opening Forward Engineering

With an EER model open:

```text
Database > Forward Engineer...
```

A wizard opens with several configuration steps.

## Step 1 - Connection Options

Select the target MySQL server connection. If you want to generate SQL only (without connecting), choose:

```text
Only Generate Script (no connection required)
```

This outputs the DDL to a `.sql` file.

## Step 2 - Options

Configure what gets included in the generated SQL:

```text
[x] Generate DROP Statements Before Each CREATE Statement
[x] Generate DROP Schema
[x] Omit Schema Qualifier in Object Names
[x] Generate USE Statements
[x] Add SHOW WARNINGS After Every DDL Statement
```

Key options:
- **DROP before CREATE** - useful for re-creating from scratch
- **Omit Schema Qualifier** - removes `schema.` prefix from object names

## Step 3 - Object Selection

Choose which objects to include in the forward engineering output:

```text
[x] Tables
[x] Views
[x] Stored Procedures
[x] Functions
[x] Triggers
```

Deselect items you do not want in the script.

## Step 4 - Review Generated SQL

Workbench shows the generated SQL before executing. Example output:

```sql
SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0;
SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0;
SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='TRADITIONAL,ALLOW_INVALID_DATES';

DROP SCHEMA IF EXISTS `ecommerce`;
CREATE SCHEMA IF NOT EXISTS `ecommerce` DEFAULT CHARACTER SET utf8mb4;
USE `ecommerce`;

CREATE TABLE IF NOT EXISTS `ecommerce`.`customers` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `email` VARCHAR(255) NOT NULL,
  `name` VARCHAR(100) NOT NULL,
  `created_at` DATETIME NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE INDEX `email_UNIQUE` (`email` ASC)
) ENGINE = InnoDB;

CREATE TABLE IF NOT EXISTS `ecommerce`.`orders` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `customer_id` INT NOT NULL,
  `total` DECIMAL(10,2) NOT NULL,
  PRIMARY KEY (`id`),
  CONSTRAINT `fk_orders_customer_id`
    FOREIGN KEY (`customer_id`) REFERENCES `ecommerce`.`customers` (`id`)
) ENGINE = InnoDB;

SET SQL_MODE=@OLD_SQL_MODE;
SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS;
SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS;
```

## Step 5 - Execute or Save

Click **Execute** to run the SQL against the connected server, or **Save to File** to export it.

## Saving the SQL Script

To save the generated script for CI/CD pipelines or version control:

```text
Step 4 > Save to File > schema_v1.sql
```

Commit `schema_v1.sql` alongside your `.mwb` model file.

## Synchronizing Model with Live Database

For iterative schema changes, use **Synchronize Model** instead of Forward Engineer:

```text
Database > Synchronize Model...
```

This compares your model with the live server and generates only the `ALTER TABLE` statements needed to bring the server in sync with the model, avoiding destructive `DROP` and `CREATE` on existing tables.

## Summary

MySQL Workbench Forward Engineering converts EER diagrams into executable DDL SQL, enabling model-driven database development. Use it to generate initial schema scripts, automate database creation in CI/CD, and maintain a visual source of truth. For incremental changes to existing databases, prefer **Synchronize Model** to generate safe `ALTER` statements instead of full `CREATE` scripts.
