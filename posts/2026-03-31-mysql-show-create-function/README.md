# How to Use SHOW CREATE FUNCTION in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Function, Stored Routine, Administration

Description: Learn how to use SHOW CREATE FUNCTION in MySQL to retrieve the full DDL of a stored function for documentation, migration, and debugging.

---

## What Is SHOW CREATE FUNCTION?

MySQL stored functions encapsulate reusable SQL logic that returns a single value. The `SHOW CREATE FUNCTION` statement retrieves the exact `CREATE FUNCTION` DDL for any existing stored function. This is useful for backups, reviewing logic, migrating to another server, or regenerating functions after schema changes.

## Basic Syntax

```sql
SHOW CREATE FUNCTION function_name;
SHOW CREATE FUNCTION db_name.function_name;
```

## Creating a Sample Function

```sql
DELIMITER $$

CREATE FUNCTION calculate_tax(price DECIMAL(10,2), rate DECIMAL(5,2))
RETURNS DECIMAL(10,2)
DETERMINISTIC
COMMENT 'Calculates tax amount given price and rate'
BEGIN
  RETURN price * (rate / 100);
END$$

DELIMITER ;
```

## Retrieving the Function DDL

```sql
SHOW CREATE FUNCTION calculate_tax\G
```

Sample output:

```text
*************************** 1. row ***************************
            Function: calculate_tax
            sql_mode: ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,...
     Create Function: CREATE DEFINER=`root`@`localhost` FUNCTION `calculate_tax`(
                        price DECIMAL(10,2),
                        rate DECIMAL(5,2)
                      ) RETURNS decimal(10,2)
                      DETERMINISTIC
                      COMMENT 'Calculates tax amount given price and rate'
                      BEGIN
                        RETURN price * (rate / 100);
                      END
character_set_client: utf8mb4
collation_connection: utf8mb4_0900_ai_ci
  Database Collation: utf8mb4_0900_ai_ci
```

## Listing All Functions First

```sql
-- List all functions in the current database
SHOW FUNCTION STATUS WHERE Db = DATABASE();

-- Query information_schema for detail
SELECT ROUTINE_NAME, ROUTINE_TYPE, DATA_TYPE, SECURITY_TYPE, DEFINER
FROM information_schema.ROUTINES
WHERE ROUTINE_TYPE = 'FUNCTION'
  AND ROUTINE_SCHEMA = 'mydb';
```

## Using the Output for Migration

When migrating to a new server:

```bash
# Export with mysqldump including routines
mysqldump --routines --no-data mydb > schema_with_routines.sql
```

Or retrieve manually and apply on the target:

```sql
-- On source
SHOW CREATE FUNCTION mydb.calculate_tax\G

-- On target: create the function using the retrieved DDL
```

## Modifying a Function

To update function logic, you must drop and recreate it:

```sql
DROP FUNCTION IF EXISTS calculate_tax;

DELIMITER $$
CREATE FUNCTION calculate_tax(price DECIMAL(10,2), rate DECIMAL(5,2))
RETURNS DECIMAL(10,2)
DETERMINISTIC
BEGIN
  DECLARE tax DECIMAL(10,2);
  SET tax = price * (rate / 100);
  RETURN ROUND(tax, 2);
END$$
DELIMITER ;
```

## Required Privileges

```sql
-- To view functions
SHOW FUNCTION STATUS;

-- To create or drop functions
GRANT CREATE ROUTINE, ALTER ROUTINE ON mydb.* TO 'dev_user'@'localhost';
```

## Summary

`SHOW CREATE FUNCTION` is the simplest way to retrieve the full definition of a MySQL stored function. Use it to audit your routines, prepare migration scripts, or rebuild a function after accidental deletion. Pair it with `SHOW FUNCTION STATUS` and `information_schema.ROUTINES` for a complete view of all stored functions in your database.
