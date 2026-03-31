# How to Use MySQL Workbench for Data Modeling (EER Diagrams)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Workbench, EER, Data Modeling, Schema

Description: Learn how to build Enhanced Entity-Relationship diagrams in MySQL Workbench to visually model database structures with entities, relationships, and cardinality.

---

## Introduction

Enhanced Entity-Relationship (EER) diagrams are the standard way to visually represent a database schema. MySQL Workbench includes a full EER diagram editor that lets you create entities (tables), define relationships (one-to-one, one-to-many, many-to-many), and generate SQL DDL from the model. This visual approach helps teams communicate schema design and catch structural issues early.

## Creating a New EER Diagram

1. Open MySQL Workbench
2. Go to **File > New Model**
3. Double-click **Add Diagram** to open the EER canvas

The canvas provides a grid workspace where you place and connect table entities.

## Adding Tables (Entities)

Click the **New Table** tool in the left toolbar (table icon) and click on the canvas to place it. A default table named `table1` appears. Double-click it to open the table editor and define:

```text
Table Name: customers

Columns:
  id         INT        PK, NN, AI
  name       VARCHAR(100)  NN
  email      VARCHAR(255)  NN, UQ
  created_at DATETIME   NN
```

Repeat for all entities in your model. For example, also create an `orders` table:

```text
Table Name: orders

Columns:
  id          INT        PK, NN, AI
  customer_id INT        NN
  total       DECIMAL(10,2)  NN
  status      VARCHAR(20)   NN
  created_at  DATETIME   NN
```

## Defining Relationships

Use the relationship tools in the left toolbar to draw relationships between tables:

- **1:1 Non-Identifying** - one-to-one, FK nullable
- **1:N Non-Identifying** - one-to-many, FK nullable
- **1:N Identifying** - one-to-many, FK is part of PK
- **N:M** - many-to-many (creates a junction table)

To create a one-to-many relationship between `customers` and `orders`:

1. Select the **1:N Non-Identifying** tool
2. Click on `orders` (the child/many side)
3. Click on `customers` (the parent/one side)

Workbench automatically adds `customer_id` to `orders` and creates the foreign key constraint.

## Reading the EER Diagram

Once relationships are drawn, the diagram shows:

```text
customers 1 ----< orders
```

Where `1` means one customer and `<` (crow's foot) means many orders. The foreign key column appears in the `orders` entity.

## Many-to-Many Relationships

For a `products` and `orders` many-to-many relationship:

1. Select the **N:M** tool
2. Click `orders`, then `products`
3. Workbench creates a junction table `orders_has_products` with foreign keys to both tables

## Adding Notes and Annotations

Right-click on the canvas and select **Place a New Note** to add documentation labels to your diagram. Use this to explain design decisions or mark areas under review.

## Exporting the Diagram

Export the EER diagram as an image:

```text
File > Export > Export as PNG
```

Or generate the SQL DDL:

```text
Database > Forward Engineer
```

The forward engineering wizard generates:

```sql
CREATE TABLE `customers` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(100) NOT NULL,
  `email` VARCHAR(255) NOT NULL,
  `created_at` DATETIME NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE INDEX `email_UNIQUE` (`email`)
) ENGINE=InnoDB;

CREATE TABLE `orders` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `customer_id` INT NOT NULL,
  `total` DECIMAL(10,2) NOT NULL,
  `status` VARCHAR(20) NOT NULL,
  `created_at` DATETIME NOT NULL,
  PRIMARY KEY (`id`),
  CONSTRAINT `fk_orders_customer_id`
    FOREIGN KEY (`customer_id`) REFERENCES `customers` (`id`)
) ENGINE=InnoDB;
```

## Summary

MySQL Workbench EER diagrams give you a visual language for database schema design. By placing tables, drawing relationships with the correct cardinality tools, and using forward engineering to generate SQL, you can design and document complex schemas in a way that developers and stakeholders can both understand. Save the `.mwb` model file in version control to track schema evolution over time.
