# How to Implement Upsert (Insert or Update) in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Upsert, Insert, Update, InnoDB

Description: Learn how to implement upsert operations in MySQL using INSERT ... ON DUPLICATE KEY UPDATE and REPLACE INTO with practical examples and trade-offs.

---

## What Is Upsert?

An upsert combines INSERT and UPDATE into a single atomic operation: insert a row if it does not exist, or update it if it does. MySQL provides two primary ways to achieve this.

## INSERT ... ON DUPLICATE KEY UPDATE

This is the recommended upsert syntax. It triggers the `UPDATE` clause when the inserted row would create a duplicate in any `UNIQUE` index or `PRIMARY KEY`.

```sql
INSERT INTO page_views (page_id, view_count, last_viewed)
VALUES (101, 1, NOW())
ON DUPLICATE KEY UPDATE
  view_count  = view_count + 1,
  last_viewed = NOW();
```

If `page_id` is a primary key, this inserts the row on first access and increments the counter on every subsequent call.

## Upsert with Multiple Columns

```sql
INSERT INTO user_settings (user_id, setting_key, setting_value, updated_at)
VALUES (42, 'theme', 'dark', NOW())
ON DUPLICATE KEY UPDATE
  setting_value = VALUES(setting_value),
  updated_at    = VALUES(updated_at);
```

The `VALUES()` function (pre-MySQL 8.0.20) references the value from the INSERT clause. In MySQL 8.0.20 and later, use aliases instead:

```sql
INSERT INTO user_settings (user_id, setting_key, setting_value, updated_at)
VALUES (42, 'theme', 'dark', NOW()) AS new_row
ON DUPLICATE KEY UPDATE
  setting_value = new_row.setting_value,
  updated_at    = new_row.updated_at;
```

## Bulk Upsert

Multi-row upserts follow the same syntax:

```sql
INSERT INTO product_stock (product_id, quantity)
VALUES
  (1, 100),
  (2, 50),
  (3, 200)
ON DUPLICATE KEY UPDATE
  quantity = VALUES(quantity);
```

This is efficient for syncing large datasets from external sources.

## Checking What Happened with ROW_COUNT()

`ROW_COUNT()` returns 1 for a new insert, 2 for an update, and 0 if the update was a no-op (same values):

```sql
INSERT INTO user_sessions (session_id, user_id, expires_at)
VALUES ('abc123', 7, DATE_ADD(NOW(), INTERVAL 1 HOUR))
ON DUPLICATE KEY UPDATE
  expires_at = VALUES(expires_at);

SELECT ROW_COUNT() AS affected;
-- 1 = inserted, 2 = updated, 0 = no change
```

## REPLACE INTO

`REPLACE INTO` is an alternative that deletes the existing row and inserts a new one when a duplicate key is found:

```sql
REPLACE INTO page_views (page_id, view_count, last_viewed)
VALUES (101, 999, NOW());
```

This causes the auto-increment `id` to change, foreign key cascades to fire, and triggers to run twice (`DELETE` then `INSERT`). Avoid `REPLACE INTO` when referential integrity or auto-increment stability matters. Prefer `ON DUPLICATE KEY UPDATE`.

## Upsert in a Transaction

When atomicity across multiple upserts is needed, wrap them in a transaction:

```sql
START TRANSACTION;

INSERT INTO orders (order_id, status) VALUES (500, 'pending')
ON DUPLICATE KEY UPDATE status = 'pending';

INSERT INTO order_items (order_id, product_id, qty)
VALUES (500, 10, 3)
ON DUPLICATE KEY UPDATE qty = VALUES(qty);

COMMIT;
```

## Summary

Use `INSERT ... ON DUPLICATE KEY UPDATE` as the standard MySQL upsert mechanism. It is atomic, efficient, and works cleanly with InnoDB locking. Reference inserted values with the new alias syntax in MySQL 8.0.20+. Avoid `REPLACE INTO` in tables with foreign keys or important auto-increment IDs. Use `ROW_COUNT()` to distinguish inserts from updates when the calling code needs to know the outcome.
