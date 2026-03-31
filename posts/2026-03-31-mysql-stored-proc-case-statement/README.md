# How to Use CASE Statement in MySQL Stored Procedures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Stored Procedure, CASE, Control Flow, SQL

Description: Learn how to use CASE statements in MySQL stored procedures for multi-branch conditional logic, including simple and searched CASE syntax.

---

The `CASE` statement in MySQL stored procedures provides multi-way branching logic. It is the procedural equivalent of `CASE` expressions in SQL queries but executes statements rather than returning values. MySQL supports two forms: simple CASE and searched CASE.

## Simple CASE Statement

The simple CASE compares a single expression against a list of values:

```sql
DELIMITER //

CREATE PROCEDURE process_order_status(IN p_order_id INT)
BEGIN
    DECLARE v_status VARCHAR(20);

    SELECT status INTO v_status FROM orders WHERE id = p_order_id;

    CASE v_status
        WHEN 'pending' THEN
            UPDATE orders SET priority = 'normal' WHERE id = p_order_id;
            INSERT INTO audit_log (action) VALUES ('Marked pending order as normal priority');

        WHEN 'processing' THEN
            UPDATE orders SET priority = 'high' WHERE id = p_order_id;

        WHEN 'shipped' THEN
            INSERT INTO notifications (order_id, message)
            VALUES (p_order_id, 'Your order has shipped!');

        WHEN 'delivered' THEN
            UPDATE orders SET completed_at = NOW() WHERE id = p_order_id;
            UPDATE customers SET order_count = order_count + 1
            WHERE id = (SELECT customer_id FROM orders WHERE id = p_order_id);

        ELSE
            SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Unknown order status';
    END CASE;
END //

DELIMITER ;
```

## Searched CASE Statement

The searched CASE evaluates a series of boolean conditions and executes the first matching branch:

```sql
DELIMITER //

CREATE PROCEDURE apply_discount(
    IN p_customer_id INT,
    IN p_order_total DECIMAL(10,2),
    OUT p_discount DECIMAL(10,2)
)
BEGIN
    DECLARE v_order_count INT;

    SELECT COUNT(*) INTO v_order_count
    FROM orders
    WHERE customer_id = p_customer_id
      AND status = 'delivered';

    CASE
        WHEN p_order_total >= 1000.00 THEN
            SET p_discount = p_order_total * 0.15;

        WHEN p_order_total >= 500.00 AND v_order_count >= 10 THEN
            SET p_discount = p_order_total * 0.12;

        WHEN p_order_total >= 500.00 THEN
            SET p_discount = p_order_total * 0.08;

        WHEN v_order_count >= 5 THEN
            SET p_discount = p_order_total * 0.05;

        ELSE
            SET p_discount = 0.00;
    END CASE;
END //

DELIMITER ;

CALL apply_discount(42, 750.00, @disc);
SELECT @disc AS discount_amount;
```

## CASE with Multiple Statements Per Branch

Each WHEN branch can contain multiple SQL statements:

```sql
DELIMITER //

CREATE PROCEDURE onboard_customer(
    IN p_customer_id INT,
    IN p_tier VARCHAR(20)
)
BEGIN
    CASE p_tier
        WHEN 'premium' THEN
            UPDATE customers SET tier = 'premium' WHERE id = p_customer_id;
            INSERT INTO perks (customer_id, perk_type) VALUES (p_customer_id, 'free_shipping');
            INSERT INTO perks (customer_id, perk_type) VALUES (p_customer_id, 'priority_support');
            INSERT INTO notifications (customer_id, message)
            VALUES (p_customer_id, 'Welcome to Premium! Enjoy free shipping.');

        WHEN 'standard' THEN
            UPDATE customers SET tier = 'standard' WHERE id = p_customer_id;
            INSERT INTO notifications (customer_id, message)
            VALUES (p_customer_id, 'Welcome to our standard plan.');

        ELSE
            UPDATE customers SET tier = 'free' WHERE id = p_customer_id;
    END CASE;

    UPDATE customers SET updated_at = NOW() WHERE id = p_customer_id;
END //

DELIMITER ;
```

## Nesting CASE Inside a Loop

```sql
DELIMITER //

CREATE PROCEDURE categorize_products()
BEGIN
    DECLARE v_done INT DEFAULT FALSE;
    DECLARE v_id INT;
    DECLARE v_price DECIMAL(10,2);
    DECLARE v_category VARCHAR(20);

    DECLARE product_cursor CURSOR FOR
        SELECT id, price FROM products WHERE category IS NULL;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    OPEN product_cursor;

    read_loop: LOOP
        FETCH product_cursor INTO v_id, v_price;
        IF v_done THEN LEAVE read_loop; END IF;

        CASE
            WHEN v_price < 10.00 THEN SET v_category = 'budget';
            WHEN v_price < 50.00 THEN SET v_category = 'mid-range';
            ELSE SET v_category = 'premium';
        END CASE;

        UPDATE products SET category = v_category WHERE id = v_id;
    END LOOP;

    CLOSE product_cursor;
END //

DELIMITER ;
```

## CASE vs. IF...THEN...ELSE

Use `CASE` when you have more than two or three branches comparing against a single variable or testing mutually exclusive conditions. Use `IF...ELSEIF` for two-branch logic or when conditions involve different variables.

## Summary

MySQL stored procedures support simple CASE (compares one expression to multiple values) and searched CASE (evaluates boolean conditions). The `ELSE` clause catches unmatched cases - omitting it causes an error if no condition matches. Each `WHEN` branch can contain multiple statements. Use searched CASE for complex discount calculations, status routing, and classification logic where conditions combine multiple variables.
