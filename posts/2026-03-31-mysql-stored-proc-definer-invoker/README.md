# How to Use DEFINER vs INVOKER Security in MySQL Stored Procedures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Stored Procedure, Security, Permission, Administration

Description: Learn the difference between DEFINER and INVOKER security modes in MySQL stored procedures and when to use each for correct privilege management.

---

MySQL stored procedures execute with one of two security contexts: `DEFINER` (default) or `INVOKER`. This setting controls whose privileges are used when the procedure accesses database objects. Choosing the right security type is critical for building secure database APIs.

## Understanding DEFINER Security

With `SQL SECURITY DEFINER`, the procedure runs with the privileges of the user who created it, regardless of who calls it. This is the default behavior:

```sql
-- Explicitly declare DEFINER security (same as default)
DELIMITER //

CREATE DEFINER='admin_user'@'%' PROCEDURE get_all_salaries()
SQL SECURITY DEFINER
BEGIN
    -- Executes with admin_user's privileges, not the caller's
    SELECT employee_id, salary, department
    FROM hr.salaries
    ORDER BY salary DESC;
END //

DELIMITER ;
```

With DEFINER mode, `app_user` can call this procedure and see salary data even if `app_user` has no `SELECT` privilege on `hr.salaries`. The procedure acts as a controlled gateway.

## Understanding INVOKER Security

With `SQL SECURITY INVOKER`, the procedure runs with the calling user's own privileges:

```sql
DELIMITER //

CREATE PROCEDURE get_my_orders()
SQL SECURITY INVOKER
BEGIN
    -- Executes with the CALLER's privileges
    -- Caller must have SELECT on orders table
    SELECT id, total, status, created_at
    FROM orders
    WHERE customer_id = (
        SELECT id FROM customers WHERE email = USER()
    );
END //

DELIMITER ;
```

If the calling user lacks `SELECT` on `orders`, this procedure fails with an access denied error.

## When to Use DEFINER

DEFINER is appropriate when you want to:
- Grant controlled access to sensitive tables without directly granting table privileges
- Create a stable API layer where applications access data only through procedures

```sql
-- Grant app users execute access only, not direct table access
GRANT EXECUTE ON PROCEDURE mydb.get_all_salaries TO 'app_user'@'%';
REVOKE SELECT ON hr.salaries FROM 'app_user'@'%';

-- app_user can call the procedure but cannot directly query the table
```

This is a common security pattern for restricting direct table access in multi-tenant applications.

## When to Use INVOKER

INVOKER is appropriate when you want the procedure to:
- Respect row-level security or view-based access restrictions
- Audit who actually performed an action (not just who owns the procedure)
- Prevent privilege escalation where callers should not exceed their own permissions

```sql
DELIMITER //

CREATE PROCEDURE update_user_profile(
    IN p_user_id INT,
    IN p_name VARCHAR(100)
)
SQL SECURITY INVOKER
BEGIN
    -- Invoker mode: caller must have UPDATE on users
    -- Prevents non-owners from updating other users' profiles via the procedure
    IF p_user_id != (SELECT id FROM users WHERE email = USER()) THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Access denied: cannot modify another user profile';
    END IF;

    UPDATE users SET name = p_name WHERE id = p_user_id;
END //

DELIMITER ;
```

## Checking Current Security Type

```sql
-- Check security type for all procedures
SELECT
    ROUTINE_NAME,
    SECURITY_TYPE,
    DEFINER
FROM information_schema.ROUTINES
WHERE ROUTINE_SCHEMA = 'mydb'
  AND ROUTINE_TYPE = 'PROCEDURE'
ORDER BY ROUTINE_NAME;
```

## Changing the Security Type

```sql
-- Change from DEFINER to INVOKER
ALTER PROCEDURE mydb.get_all_salaries SQL SECURITY INVOKER;

-- Change back to DEFINER
ALTER PROCEDURE mydb.get_all_salaries SQL SECURITY DEFINER;
```

## Security Risks to Avoid

A DEFINER procedure created by a superuser that accepts unvalidated input can become a privilege escalation vector:

```sql
-- DANGEROUS: DEFINER procedure with SQL injection risk
CREATE PROCEDURE run_search(IN p_table VARCHAR(100))
SQL SECURITY DEFINER
BEGIN
    -- Never do this: dynamic SQL with user input
    SET @sql = CONCAT('SELECT * FROM ', p_table);
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
END;
```

Validate and sanitize all inputs in DEFINER procedures because they execute with elevated privileges.

## Summary

`SQL SECURITY DEFINER` (the default) runs the procedure with the creator's privileges - useful for controlled access to sensitive tables without granting direct privileges. `SQL SECURITY INVOKER` uses the caller's own privileges - appropriate when callers should not exceed their permissions. Grant `EXECUTE` on DEFINER procedures to restrict application access to specific operations. Always validate inputs in DEFINER procedures to prevent privilege escalation.
