# How to Use SHOW CREATE PROCEDURE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Procedure, Stored Routine, Administration

Description: Learn how to use SHOW CREATE PROCEDURE in MySQL to extract the full DDL of a stored procedure for review, migration, or documentation.

---

## What Is SHOW CREATE PROCEDURE?

Stored procedures in MySQL are named blocks of SQL that can accept parameters, execute logic, and return result sets. The `SHOW CREATE PROCEDURE` statement retrieves the complete `CREATE PROCEDURE` DDL for any existing procedure. This is essential for auditing, version control, and schema migration workflows.

## Basic Syntax

```sql
SHOW CREATE PROCEDURE procedure_name;
SHOW CREATE PROCEDURE db_name.procedure_name;
```

## Creating a Sample Procedure

```sql
DELIMITER $$

CREATE PROCEDURE get_user_orders(IN user_id INT, OUT order_count INT)
COMMENT 'Returns total number of orders for a given user'
BEGIN
  SELECT COUNT(*) INTO order_count
  FROM orders
  WHERE customer_id = user_id;
END$$

DELIMITER ;
```

## Retrieving the Procedure DDL

```sql
SHOW CREATE PROCEDURE get_user_orders\G
```

Sample output:

```text
*************************** 1. row ***************************
           Procedure: get_user_orders
            sql_mode: ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,...
    Create Procedure: CREATE DEFINER=`root`@`localhost` PROCEDURE `get_user_orders`(
                        IN user_id INT,
                        OUT order_count INT
                      )
                      COMMENT 'Returns total number of orders for a given user'
                      BEGIN
                        SELECT COUNT(*) INTO order_count
                        FROM orders
                        WHERE customer_id = user_id;
                      END
character_set_client: utf8mb4
collation_connection: utf8mb4_0900_ai_ci
  Database Collation: utf8mb4_0900_ai_ci
```

## Listing Stored Procedures

```sql
-- List all procedures in the current database
SHOW PROCEDURE STATUS WHERE Db = DATABASE();

-- Use information_schema for richer detail
SELECT ROUTINE_NAME, DEFINER, CREATED, LAST_ALTERED, SECURITY_TYPE
FROM information_schema.ROUTINES
WHERE ROUTINE_TYPE = 'PROCEDURE'
  AND ROUTINE_SCHEMA = 'mydb'
ORDER BY ROUTINE_NAME;
```

## Executing a Procedure

```sql
-- Call the procedure
CALL get_user_orders(42, @count);
SELECT @count;
```

## Exporting Procedures for Migration

```bash
# Export all routines with mysqldump
mysqldump --routines --no-data --no-create-info mydb > routines.sql
```

Or export individually using the output of `SHOW CREATE PROCEDURE` and save it to a `.sql` file.

## Updating a Procedure

MySQL does not support `ALTER PROCEDURE` for body changes. Drop and recreate:

```sql
DROP PROCEDURE IF EXISTS get_user_orders;

DELIMITER $$
CREATE PROCEDURE get_user_orders(IN user_id INT, OUT order_count INT)
BEGIN
  SELECT COUNT(*) INTO order_count
  FROM orders
  WHERE customer_id = user_id AND status != 'cancelled';
END$$
DELIMITER ;
```

## Required Privileges

```sql
-- Grant routine privileges to a user
GRANT EXECUTE ON mydb.* TO 'app_user'@'localhost';
GRANT CREATE ROUTINE, ALTER ROUTINE ON mydb.* TO 'dba_user'@'localhost';
```

## Summary

`SHOW CREATE PROCEDURE` makes it easy to retrieve the full definition of a stored procedure in MySQL. Use it for version-controlling your stored routines, auditing logic before a deployment, or recreating procedures after a migration. Combine with `SHOW PROCEDURE STATUS` to discover all procedures and their metadata in a given schema.
