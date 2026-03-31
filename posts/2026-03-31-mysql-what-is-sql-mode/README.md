# What Is SQL Mode in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL Mode, Configuration, Data Validation, Strict Mode

Description: SQL mode in MySQL is a server configuration that controls syntax and data validation behavior, allowing MySQL to operate in stricter or more permissive modes.

---

## Overview

SQL mode is a MySQL server setting that controls how MySQL handles SQL syntax and data validation. By enabling or disabling specific SQL mode flags, you can make MySQL behave like other databases (for portability), enforce strict data validation, or relax constraints to allow legacy data to be imported. SQL mode can be set globally (for all connections) or at the session level (for a single connection).

## Viewing the Current SQL Mode

```sql
-- Current session SQL mode
SELECT @@sql_mode;

-- Global SQL mode
SELECT @@GLOBAL.sql_mode;

-- Via SHOW VARIABLES
SHOW VARIABLES LIKE 'sql_mode';
```

## Setting SQL Mode

```sql
-- Set for the current session only
SET SESSION sql_mode = 'STRICT_TRANS_TABLES,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO';

-- Set globally for all new connections
SET GLOBAL sql_mode = 'STRICT_TRANS_TABLES,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO';
```

To persist across restarts, add to `my.cnf`:

```ini
[mysqld]
sql_mode = STRICT_TRANS_TABLES,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
```

## Default SQL Mode (MySQL 8.0)

MySQL 8.0 enables a strict default:

```
STRICT_TRANS_TABLES, NO_ENGINE_SUBSTITUTION
```

MySQL 5.7 adds additional defaults:
```
ONLY_FULL_GROUP_BY, STRICT_TRANS_TABLES, NO_ZERO_IN_DATE, NO_ZERO_DATE,
ERROR_FOR_DIVISION_BY_ZERO, NO_AUTO_CREATE_USER, NO_ENGINE_SUBSTITUTION
```

## Key SQL Mode Flags

**STRICT_TRANS_TABLES**: Rejects invalid data for transactional tables (InnoDB). Without it, MySQL silently inserts adjusted values and only issues a warning.

```sql
SET SESSION sql_mode = 'STRICT_TRANS_TABLES';
INSERT INTO orders (total) VALUES ('not_a_number');
-- ERROR: Incorrect decimal value

SET SESSION sql_mode = '';
INSERT INTO orders (total) VALUES ('not_a_number');
-- Inserts 0 with a warning -- silent data corruption!
```

**ONLY_FULL_GROUP_BY**: Requires that SELECT columns either appear in GROUP BY or use an aggregate function.

```sql
SET SESSION sql_mode = 'ONLY_FULL_GROUP_BY';
SELECT name, SUM(total) FROM orders GROUP BY customer_id;
-- ERROR: name is not in GROUP BY and not an aggregate
```

**NO_ZERO_DATE / NO_ZERO_IN_DATE**: Reject dates like `'0000-00-00'` or `'2026-00-01'`.

**ERROR_FOR_DIVISION_BY_ZERO**: Return an error instead of NULL when dividing by zero.

```sql
SET SESSION sql_mode = 'ERROR_FOR_DIVISION_BY_ZERO';
SELECT 10 / 0;
-- ERROR: Division by 0
```

**ANSI_QUOTES**: Treat `"` as identifier quotes (like standard SQL) instead of string quotes.

**NO_ENGINE_SUBSTITUTION**: Error if the requested storage engine is unavailable, instead of silently substituting the default.

## Combining Modes

SQL mode is a comma-separated list. You can combine any modes:

```sql
SET sql_mode = 'STRICT_TRANS_TABLES,ONLY_FULL_GROUP_BY,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO';
```

Use `TRADITIONAL` as a shorthand for the strict bundle:

```sql
SET sql_mode = 'TRADITIONAL';
-- Equivalent to a set of strict modes
```

## Summary

SQL mode in MySQL determines how strictly the server validates data and SQL syntax. Strict modes like `STRICT_TRANS_TABLES` prevent silent data truncation and invalid value insertion. Other modes control GROUP BY semantics, zero-date handling, and identifier quoting. Understanding and configuring SQL mode appropriately prevents subtle data corruption bugs and ensures consistent behavior across development, staging, and production environments.
