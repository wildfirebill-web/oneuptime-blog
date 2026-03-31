# How to Fix ERROR 1418 Function Has None of DETERMINISTIC in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Error, Stored Function, Binary Log, Troubleshooting

Description: Learn how to fix MySQL ERROR 1418 when creating stored functions by declaring them DETERMINISTIC or adjusting log_bin_trust_function_creators.

---

## What Is ERROR 1418?

```
ERROR 1418 (HY000): This function has none of DETERMINISTIC, NO SQL, or READS SQL DATA
in its declaration and binary logging is enabled (you *might* want to use the less safe
log_bin_trust_function_creators variable)
```

This error occurs when you try to create a stored function without explicitly declaring one of these characteristics, and binary logging (`log_bin`) is enabled. MySQL requires the declaration to ensure safe binary log-based replication.

## Why Does This Happen?

When binary logging is on, MySQL must know whether a function is safe to replicate. If a function is non-deterministic (returns different results for the same inputs), replication could produce different results on replicas. MySQL blocks creation of such functions unless you explicitly acknowledge this risk.

## Fix 1: Declare the Function as DETERMINISTIC

If your function always returns the same output for the same inputs (no `NOW()`, `RAND()`, `UUID()`, etc.), declare it `DETERMINISTIC`:

```sql
DELIMITER //

CREATE FUNCTION calculate_tax(price DECIMAL(10,2))
RETURNS DECIMAL(10,2)
DETERMINISTIC
BEGIN
  RETURN price * 0.20;
END //

DELIMITER ;
```

## Fix 2: Use NO SQL or READS SQL DATA

If the function does not use SQL at all:

```sql
DELIMITER //

CREATE FUNCTION double_value(n INT)
RETURNS INT
NO SQL
BEGIN
  RETURN n * 2;
END //

DELIMITER ;
```

If the function reads data but does not modify it:

```sql
DELIMITER //

CREATE FUNCTION get_user_name(user_id INT)
RETURNS VARCHAR(100)
READS SQL DATA
BEGIN
  DECLARE uname VARCHAR(100);
  SELECT name INTO uname FROM users WHERE id = user_id;
  RETURN uname;
END //

DELIMITER ;
```

## Fix 3: Set log_bin_trust_function_creators (Temporary Fix)

If you cannot modify the function declaration (for example, it uses `NOW()` or `RAND()` and is genuinely non-deterministic), you can bypass the check:

```sql
SET GLOBAL log_bin_trust_function_creators = 1;
```

This allows creation of functions without explicit characteristics when binary logging is enabled. Use with caution on replication setups, as it may cause replica divergence.

To make it permanent, add to `my.cnf`:

```ini
[mysqld]
log_bin_trust_function_creators = 1
```

## Choosing the Right Characteristic

| Characteristic | When to Use |
|---|---|
| `DETERMINISTIC` | Function always returns same result for same inputs |
| `NO SQL` | Function contains no SQL statements |
| `READS SQL DATA` | Function reads data with SELECT but does not modify it |
| `MODIFIES SQL DATA` | Function inserts/updates/deletes data |
| `NOT DETERMINISTIC` | Result may vary (e.g., uses RAND(), NOW()) |

## Example: Fixing a Real-World Function

Before fix (fails with ERROR 1418):

```sql
DELIMITER //
CREATE FUNCTION greet_user(user_id INT)
RETURNS VARCHAR(200)
BEGIN
  DECLARE uname VARCHAR(100);
  SELECT name INTO uname FROM users WHERE id = user_id;
  RETURN CONCAT('Hello, ', uname);
END //
DELIMITER ;
```

After fix (add `READS SQL DATA`):

```sql
DELIMITER //
CREATE FUNCTION greet_user(user_id INT)
RETURNS VARCHAR(200)
READS SQL DATA
BEGIN
  DECLARE uname VARCHAR(100);
  SELECT name INTO uname FROM users WHERE id = user_id;
  RETURN CONCAT('Hello, ', uname);
END //
DELIMITER ;
```

## Summary

ERROR 1418 is MySQL's safety check for stored functions when binary logging is enabled. Fix it by adding the appropriate characteristic (`DETERMINISTIC`, `NO SQL`, or `READS SQL DATA`) to the function declaration. Use `log_bin_trust_function_creators = 1` only when you need to allow genuinely non-deterministic functions and understand the replication implications.
