# How to Pass a List of Values to a MySQL Stored Procedure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Stored Procedure, String, JSON, Dynamic SQL

Description: MySQL lacks native array parameters. Learn three practical patterns to pass a list of values to a stored procedure using strings, JSON, or temporary tables.

---

MySQL stored procedures do not support array or list parameters natively. When you need to pass a variable number of values - such as a list of IDs to delete - you have to use one of several workarounds. The three most common are: a delimited string combined with `FIND_IN_SET`, a JSON array parsed with `JSON_TABLE`, and a pre-populated temporary table.

## Option 1 - Delimited String with FIND_IN_SET

Pass a comma-separated string and use `FIND_IN_SET` to filter rows:

```sql
DELIMITER //

CREATE PROCEDURE delete_users_by_ids(IN p_id_list TEXT)
BEGIN
    DELETE FROM users
    WHERE FIND_IN_SET(id, p_id_list) > 0;
END //

DELIMITER ;
```

Call it with a comma-separated list:

```sql
CALL delete_users_by_ids('1,5,9,23');
```

This is simple but does not use indexes efficiently on large tables because `FIND_IN_SET` performs a full scan.

## Option 2 - JSON Array with JSON_TABLE (MySQL 8.0+)

JSON arrays are structured and allow index-friendly operations through `JSON_TABLE`:

```sql
DELIMITER //

CREATE PROCEDURE process_product_ids(IN p_ids JSON)
BEGIN
    SELECT p.product_id, p.name, p.price
    FROM products p
    INNER JOIN JSON_TABLE(
        p_ids,
        '$[*]' COLUMNS (val INT PATH '$')
    ) AS jt ON p.product_id = jt.val;
END //

DELIMITER ;
```

Call it:

```sql
CALL process_product_ids('[10, 20, 35, 47]');
```

`JSON_TABLE` materializes the array as a derived table the optimizer can join against, which can use indexes on `product_id`.

## Option 3 - Temporary Table

For complex operations or when the list is large, populate a temporary table before calling the procedure:

```sql
-- Caller creates the temp table
CREATE TEMPORARY TABLE tmp_target_ids (id INT NOT NULL);
INSERT INTO tmp_target_ids VALUES (3), (7), (12), (99);

DELIMITER //

CREATE PROCEDURE archive_orders_from_temp()
BEGIN
    INSERT INTO orders_archive
    SELECT * FROM orders
    WHERE order_id IN (SELECT id FROM tmp_target_ids);

    DELETE FROM orders
    WHERE order_id IN (SELECT id FROM tmp_target_ids);
END //

DELIMITER ;

CALL archive_orders_from_temp();

DROP TEMPORARY TABLE tmp_target_ids;
```

The procedure itself takes no list parameter - the caller owns the temporary table, which persists for the session.

## Choosing the Right Approach

```text
Method            | Index-friendly | MySQL version | Complexity
------------------|---------------|---------------|----------
FIND_IN_SET       | No            | All           | Low
JSON_TABLE        | Yes (join)    | 8.0+          | Medium
Temporary table   | Yes           | All           | Medium
```

For small lists (under 100 items) the delimited string approach is fine. For larger datasets or performance-sensitive queries, prefer `JSON_TABLE` on MySQL 8.0+ or the temporary table pattern.

## Summary

MySQL stored procedures cannot accept arrays directly, but three reliable patterns exist: comma-delimited strings with `FIND_IN_SET`, JSON arrays parsed by `JSON_TABLE`, and caller-managed temporary tables. Choose based on your MySQL version, expected list size, and whether index use is critical.
