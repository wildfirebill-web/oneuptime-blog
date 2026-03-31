# How to Use SHOW DATABASES in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Database, Administration

Description: Learn how to use SHOW DATABASES in MySQL to list available databases, filter results with LIKE and WHERE, and control visibility with privileges.

---

## What Is SHOW DATABASES

`SHOW DATABASES` lists all databases accessible to the current MySQL user. It is one of the most fundamental MySQL commands used to explore server contents, verify database creation, and check what is available before connecting to a specific database.

```sql
SHOW DATABASES;
SHOW SCHEMAS;  -- synonym
```

## Basic Usage

```sql
SHOW DATABASES;
```

```text
+--------------------+
| Database           |
+--------------------+
| information_schema |
| myapp_db           |
| myapp_test         |
| reporting_db       |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```

The output always includes system databases (`mysql`, `information_schema`, `performance_schema`, `sys`) for users with appropriate privileges.

## Filtering with LIKE

Use a pattern to find specific databases:

```sql
-- Find databases starting with 'myapp'
SHOW DATABASES LIKE 'myapp%';

-- Find databases containing 'test'
SHOW DATABASES LIKE '%test%';
```

```text
+---------------------+
| Database (myapp%)   |
+---------------------+
| myapp_db            |
| myapp_test          |
| myapp_staging       |
+---------------------+
```

## Filtering with WHERE

Use a `WHERE` clause for more complex filtering:

```sql
-- Equivalent to LIKE but with more flexibility
SHOW DATABASES WHERE `Database` LIKE 'app_%';
```

## Privilege-Based Visibility

By default, `SHOW DATABASES` only lists databases for which the current user has at least one privilege. Users without the `SHOW DATABASES` global privilege only see databases they have access to:

```sql
-- User with SELECT only on myapp_db sees:
SHOW DATABASES;
-- Returns only: myapp_db, information_schema
```

Granting `SHOW DATABASES` globally shows all databases:

```sql
GRANT SHOW DATABASES ON *.* TO 'developer'@'%';
```

## Using information_schema

For programmatic access, query `information_schema.SCHEMATA`:

```sql
SELECT SCHEMA_NAME, DEFAULT_CHARACTER_SET_NAME, DEFAULT_COLLATION_NAME
FROM information_schema.SCHEMATA
WHERE SCHEMA_NAME NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
ORDER BY SCHEMA_NAME;
```

This provides the same list as `SHOW DATABASES` but with additional character set and collation details, and is easier to use in scripts and stored procedures.

## Checking if a Database Exists

A common pattern in scripts is to check whether a database exists before creating or dropping it:

```bash
# Shell script check
DB_EXISTS=$(mysql -u root -p"$PASS" -e "SHOW DATABASES LIKE 'myapp_db';" | grep myapp_db)
if [ -n "$DB_EXISTS" ]; then
  echo "Database exists"
fi
```

In SQL:

```sql
SELECT COUNT(*)
FROM information_schema.SCHEMATA
WHERE SCHEMA_NAME = 'myapp_db';
```

## Counting Total Databases

```sql
SELECT COUNT(*) AS database_count
FROM information_schema.SCHEMATA
WHERE SCHEMA_NAME NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys');
```

## Identifying Databases by Size

Combine with `information_schema.TABLES` to find the largest databases:

```sql
SELECT TABLE_SCHEMA AS database_name,
       ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_size_mb
FROM information_schema.TABLES
GROUP BY TABLE_SCHEMA
ORDER BY total_size_mb DESC;
```

## Summary

`SHOW DATABASES` is the quickest way to list MySQL databases visible to the current user. Use `LIKE` for simple pattern matching, and query `information_schema.SCHEMATA` for programmatic or enriched access. Note that visibility depends on user privileges - grant `SHOW DATABASES` globally to allow a user to see all databases regardless of individual table-level grants.
