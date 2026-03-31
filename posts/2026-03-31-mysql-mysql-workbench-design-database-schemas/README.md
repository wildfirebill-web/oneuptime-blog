# How to Use MySQL Workbench to Design Database Schemas

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Workbench, Schema, Design, Database

Description: Learn how to use MySQL Workbench to visually design database schemas, create tables with relationships, and generate SQL from your models.

---

## Introduction

MySQL Workbench is a visual database design tool that allows you to create, modify, and manage database schemas through a graphical interface. The Schema Design module lets you build EER (Enhanced Entity-Relationship) diagrams, define tables, columns, indexes, and foreign keys visually, and then generate the corresponding SQL or apply changes directly to a live server.

## Creating a New Model

1. Open MySQL Workbench
2. Click **File > New Model**
3. A new EER Diagram canvas opens with a default schema named `mydb`

Double-click the schema name to rename it:

```text
Schema: ecommerce
```

## Adding Tables

Click the **Add Table** button in the toolbar or double-click on the canvas. A table editor opens at the bottom of the screen with tabs for:

- **Columns** - define column names, data types, and constraints
- **Indexes** - add indexes for performance
- **Foreign Keys** - define relationships
- **Triggers** - add row-level triggers
- **Partitioning** - configure table partitioning

## Defining Columns

In the Columns tab, define your table structure:

```text
Column Name     Data Type       Flags
-----------     ---------       -----
id              INT             PK, NN, AI, UQ
email           VARCHAR(255)    NN, UQ
created_at      DATETIME        NN
status          ENUM(...)       NN
```

Flags explained:
- `PK` - Primary Key
- `NN` - Not Null
- `AI` - Auto Increment
- `UQ` - Unique

## Creating Relationships

To create a foreign key relationship between tables, use the **1:N Relationship** tool in the sidebar and click from the child table to the parent table. Workbench automatically adds the foreign key column and constraint.

Alternatively, define it manually in the Foreign Keys tab of the child table:

```text
FK Name:           fk_orders_customer_id
Referenced Table:  ecommerce.customers
Column:            customer_id -> id
On Delete:         CASCADE
On Update:         RESTRICT
```

## Adding Indexes

In the Indexes tab, add indexes beyond the primary key:

```text
Index Name:    idx_orders_status
Type:          INDEX
Columns:       status, created_at
```

## Generating SQL from the Model

Once your diagram is complete, generate SQL with **Database > Forward Engineer**:

```sql
-- Workbench generates DDL like:
CREATE TABLE `ecommerce`.`orders` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `customer_id` INT NOT NULL,
  `total` DECIMAL(10,2) NOT NULL,
  `status` ENUM('pending', 'shipped', 'delivered') NOT NULL,
  `created_at` DATETIME NOT NULL,
  PRIMARY KEY (`id`),
  INDEX `idx_orders_status` (`status`, `created_at`),
  CONSTRAINT `fk_orders_customer_id`
    FOREIGN KEY (`customer_id`) REFERENCES `customers` (`id`)
    ON DELETE CASCADE ON UPDATE RESTRICT
) ENGINE = InnoDB;
```

## Saving the Model

Save the model as a `.mwb` file for version control:

```text
File > Save Model As > ecommerce_schema.mwb
```

Models can be committed to Git alongside application code for schema versioning.

## Applying Changes to a Live Server

Use **Database > Synchronize Model** to compare the model with a live database and generate an ALTER script for the differences.

## Summary

MySQL Workbench's schema design tools make it easy to visually create and manage database schemas without writing DDL by hand. By combining the EER diagram editor with Forward Engineering and Synchronize Model, you can maintain a visual source of truth for your database structure and safely apply changes to production servers.
