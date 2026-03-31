# How to Create a Table in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, DDL, Schema Design, SQL

Description: Learn how to create tables in MySQL using CREATE TABLE with data types, constraints, primary keys, and storage engine options.

---

## Basic CREATE TABLE Syntax

```sql
CREATE TABLE table_name (
    column1 datatype [constraints],
    column2 datatype [constraints],
    ...
    [table_constraints]
);
```

## Creating a Simple Table

```sql
CREATE TABLE users (
    id         INT          NOT NULL AUTO_INCREMENT,
    username   VARCHAR(50)  NOT NULL,
    email      VARCHAR(100) NOT NULL,
    created_at DATETIME     DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id)
);
```

## Common MySQL Data Types

```sql
-- Integer types
TINYINT     -- -128 to 127
SMALLINT    -- -32768 to 32767
INT         -- -2147483648 to 2147483647
BIGINT      -- very large integers

-- Decimal/float
DECIMAL(10, 2)  -- exact numeric, 10 digits total, 2 decimal places
FLOAT           -- approximate single precision
DOUBLE          -- approximate double precision

-- String types
CHAR(n)         -- fixed length, padded with spaces
VARCHAR(n)      -- variable length, up to n chars
TEXT            -- up to 65535 chars
MEDIUMTEXT      -- up to 16MB
LONGTEXT        -- up to 4GB

-- Date and time
DATE            -- YYYY-MM-DD
TIME            -- HH:MM:SS
DATETIME        -- YYYY-MM-DD HH:MM:SS
TIMESTAMP       -- auto-updated timestamp
YEAR            -- 4-digit year

-- Binary
BLOB            -- binary large object
JSON            -- JSON document (MySQL 5.7.8+)
BOOLEAN / BOOL  -- synonym for TINYINT(1)
```

## Table with Multiple Constraints

```sql
CREATE TABLE products (
    id           INT          NOT NULL AUTO_INCREMENT,
    sku          VARCHAR(50)  NOT NULL,
    name         VARCHAR(200) NOT NULL,
    price        DECIMAL(10,2) NOT NULL,
    stock        INT          NOT NULL DEFAULT 0,
    category_id  INT,
    created_at   TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
    updated_at   TIMESTAMP    DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uq_sku (sku),
    CONSTRAINT chk_price CHECK (price >= 0),
    CONSTRAINT chk_stock CHECK (stock >= 0)
);
```

## CREATE TABLE IF NOT EXISTS

Prevents an error if the table already exists:

```sql
CREATE TABLE IF NOT EXISTS sessions (
    session_id VARCHAR(128) NOT NULL,
    user_id    INT          NOT NULL,
    expires_at DATETIME     NOT NULL,
    PRIMARY KEY (session_id)
);
```

## Specifying the Storage Engine and Charset

```sql
CREATE TABLE logs (
    id         BIGINT       NOT NULL AUTO_INCREMENT,
    level      VARCHAR(10)  NOT NULL,
    message    TEXT         NOT NULL,
    logged_at  DATETIME     DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

`InnoDB` is the default and recommended engine for transactional workloads. `utf8mb4` supports the full Unicode character set including emoji.

## Table with Foreign Key

```sql
CREATE TABLE orders (
    id         INT  NOT NULL AUTO_INCREMENT,
    user_id    INT  NOT NULL,
    total      DECIMAL(10,2) NOT NULL,
    status     ENUM('pending', 'shipped', 'delivered') DEFAULT 'pending',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    CONSTRAINT fk_orders_user FOREIGN KEY (user_id)
        REFERENCES users (id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);
```

## Adding Indexes at Table Creation

```sql
CREATE TABLE articles (
    id          INT          NOT NULL AUTO_INCREMENT,
    title       VARCHAR(255) NOT NULL,
    author_id   INT          NOT NULL,
    published   BOOLEAN      DEFAULT FALSE,
    created_at  DATE,
    PRIMARY KEY (id),
    INDEX idx_author (author_id),
    INDEX idx_published_date (published, created_at)
);
```

## Viewing the Created Table Structure

```sql
DESCRIBE users;

-- Or
SHOW COLUMNS FROM users;

-- Or full CREATE statement
SHOW CREATE TABLE users;
```

## Summary

`CREATE TABLE` in MySQL defines a table's structure including column names, data types, constraints, indexes, and storage options. Use `IF NOT EXISTS` to avoid errors on re-runs, specify `InnoDB` as the engine for transactional support, and use `utf8mb4` for full Unicode compatibility. Constraints like `PRIMARY KEY`, `UNIQUE`, `FOREIGN KEY`, and `CHECK` enforce data integrity at the database level.
