# How to Use INSERT ... ON DUPLICATE KEY UPDATE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Upsert, INSERT ON DUPLICATE KEY UPDATE, SQL, Data Integrity

Description: Learn how to use INSERT ... ON DUPLICATE KEY UPDATE in MySQL to insert new rows or update existing ones when a unique or primary key conflict occurs.

---

## What Is INSERT ... ON DUPLICATE KEY UPDATE?

`INSERT ... ON DUPLICATE KEY UPDATE` is MySQL's upsert operation. When an INSERT would violate a `PRIMARY KEY` or `UNIQUE` constraint, instead of raising an error, MySQL updates the existing row with the specified values. If there is no conflict, the row is inserted normally.

```sql
-- Insert or update a product's stock count
INSERT INTO product_inventory (product_id, stock_count)
VALUES (101, 50)
ON DUPLICATE KEY UPDATE stock_count = VALUES(stock_count);
```

## Basic Setup and Example

```sql
CREATE TABLE page_views (
    page_id INT NOT NULL,
    view_date DATE NOT NULL,
    view_count INT DEFAULT 0,
    PRIMARY KEY (page_id, view_date)
);

-- First call: inserts the row
INSERT INTO page_views (page_id, view_date, view_count)
VALUES (1, CURDATE(), 1)
ON DUPLICATE KEY UPDATE view_count = view_count + 1;

-- Second call: updates the existing row (view_count becomes 2)
INSERT INTO page_views (page_id, view_date, view_count)
VALUES (1, CURDATE(), 1)
ON DUPLICATE KEY UPDATE view_count = view_count + 1;
```

## Using VALUES() Function

`VALUES(col)` refers to the value that would have been inserted:

```sql
INSERT INTO product_prices (product_id, price, updated_at)
VALUES (42, 19.99, NOW())
ON DUPLICATE KEY UPDATE
    price = VALUES(price),
    updated_at = VALUES(updated_at);
```

In MySQL 8.0.20+, the `VALUES()` function in `ON DUPLICATE KEY UPDATE` is deprecated. Use row aliases instead:

```sql
-- MySQL 8.0.20+ preferred syntax with row alias
INSERT INTO product_prices (product_id, price, updated_at)
VALUES (42, 19.99, NOW()) AS new_row
ON DUPLICATE KEY UPDATE
    price = new_row.price,
    updated_at = new_row.updated_at;
```

## Bulk Upsert

```sql
-- Insert multiple rows with duplicate handling
INSERT INTO inventory (sku, quantity, warehouse)
VALUES
    ('SKU001', 100, 'A1'),
    ('SKU002', 250, 'B3'),
    ('SKU003', 75, 'A2')
ON DUPLICATE KEY UPDATE
    quantity = VALUES(quantity),
    warehouse = VALUES(warehouse);
```

## Incrementing Counters

A common pattern is to increment a counter on conflict:

```sql
-- Track login count per user per day
CREATE TABLE daily_logins (
    user_id INT NOT NULL,
    login_date DATE NOT NULL,
    login_count INT DEFAULT 0,
    PRIMARY KEY (user_id, login_date)
);

INSERT INTO daily_logins (user_id, login_date, login_count)
VALUES (123, CURDATE(), 1)
ON DUPLICATE KEY UPDATE login_count = login_count + 1;
```

## Upsert with Timestamps

```sql
CREATE TABLE user_profiles (
    user_id INT PRIMARY KEY,
    display_name VARCHAR(100),
    bio TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Insert or update profile data
INSERT INTO user_profiles (user_id, display_name, bio)
VALUES (456, 'Alice J.', 'Software engineer and hiker')
ON DUPLICATE KEY UPDATE
    display_name = VALUES(display_name),
    bio = VALUES(bio);
-- updated_at is automatically refreshed on update by ON UPDATE CURRENT_TIMESTAMP
```

## Checking if a Row Was Inserted or Updated

MySQL returns different affected_rows values:

```text
- 1: row was inserted
- 2: row existed and was updated
- 0: row existed but UPDATE set the same values (no change)
```

```sql
-- After executing the statement, check ROW_COUNT()
INSERT INTO inventory (sku, quantity)
VALUES ('SKU001', 100)
ON DUPLICATE KEY UPDATE quantity = VALUES(quantity);

SELECT ROW_COUNT();
-- Returns 1 if inserted, 2 if updated, 0 if no change
```

## Important Behavior Notes

```sql
-- AUTO_INCREMENT is incremented even when UPDATE is triggered
-- This can cause gaps in your AUTO_INCREMENT sequence
CREATE TABLE events (
    id INT AUTO_INCREMENT PRIMARY KEY,
    event_name VARCHAR(100) UNIQUE,
    event_count INT DEFAULT 0
);

INSERT INTO events (event_name, event_count)
VALUES ('page_load', 1)
ON DUPLICATE KEY UPDATE event_count = event_count + 1;
-- AUTO_INCREMENT counter advances even on duplicates
```

## Summary

`INSERT ... ON DUPLICATE KEY UPDATE` is MySQL's built-in upsert mechanism that elegantly handles insert-or-update logic in a single atomic statement. It is ideal for counter increments, cache updates, and syncing data where the record may or may not exist. In MySQL 8.0.20+, use the row alias syntax instead of `VALUES()`. Remember that `AUTO_INCREMENT` advances even on duplicate key updates, which may cause gaps.
