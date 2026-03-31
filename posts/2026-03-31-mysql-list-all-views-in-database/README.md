# How to List All Views in a MySQL Database

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, View, Information Schema, SQL, Database Administration

Description: Learn how to list all views in a MySQL database using SHOW FULL TABLES, information_schema.VIEWS, and filter by schema or updatability.

---

## Why List Views?

As a MySQL database grows, teams create many views for reporting, security, and query simplification. Knowing how to enumerate all views in a schema helps with auditing, documentation, and cleanup of stale objects.

## Method 1 - SHOW FULL TABLES

`SHOW FULL TABLES` returns both base tables and views, along with a `Table_type` column:

```sql
SHOW FULL TABLES IN mydb WHERE Table_type = 'VIEW';
```

This is the fastest approach when connected to the target database:

```sql
USE mydb;
SHOW FULL TABLES WHERE Table_type = 'VIEW';
```

Sample output:

```text
+------------------+------------+
| Tables_in_mydb   | Table_type |
+------------------+------------+
| active_employees | VIEW       |
| dept_summary     | VIEW       |
| order_counts     | VIEW       |
+------------------+------------+
```

## Method 2 - information_schema.VIEWS

For richer metadata, query `information_schema.VIEWS`:

```sql
SELECT TABLE_NAME,
       DEFINER,
       IS_UPDATABLE,
       CHECK_OPTION,
       SECURITY_TYPE
FROM information_schema.VIEWS
WHERE TABLE_SCHEMA = 'mydb'
ORDER BY TABLE_NAME;
```

This reveals who owns the view (`DEFINER`), whether it is updatable, whether `WITH CHECK OPTION` is set, and whether it runs with `DEFINER` or `INVOKER` security.

## Method 3 - List Views Across All Databases

```sql
SELECT TABLE_SCHEMA, TABLE_NAME, IS_UPDATABLE
FROM information_schema.VIEWS
ORDER BY TABLE_SCHEMA, TABLE_NAME;
```

This is useful for a cross-schema audit on a server hosting multiple databases.

## Filtering Views by Property

Find all non-updatable views:

```sql
SELECT TABLE_SCHEMA, TABLE_NAME
FROM information_schema.VIEWS
WHERE TABLE_SCHEMA = 'mydb'
  AND IS_UPDATABLE = 'NO';
```

Find views with a specific definer:

```sql
SELECT TABLE_NAME, DEFINER
FROM information_schema.VIEWS
WHERE TABLE_SCHEMA = 'mydb'
  AND DEFINER = 'app_user@%';
```

## Listing Views from the Command Line

```bash
mysql -u root -p mydb -e "SHOW FULL TABLES WHERE Table_type = 'VIEW';"
```

Or for a quick count:

```bash
mysql -u root -p -e \
  "SELECT COUNT(*) FROM information_schema.VIEWS WHERE TABLE_SCHEMA='mydb';"
```

## Exporting View Definitions

To review the full definition of every view in a schema, iterate through the names:

```sql
SELECT CONCAT('SHOW CREATE VIEW ', TABLE_NAME, ';') AS cmd
FROM information_schema.VIEWS
WHERE TABLE_SCHEMA = 'mydb';
```

Or use `mysqldump` to export only views:

```bash
mysqldump --no-data mydb | grep -A 20 'CREATE.*VIEW'
```

## Summary

Use `SHOW FULL TABLES WHERE Table_type = 'VIEW'` for a quick listing within a database, and query `information_schema.VIEWS` when you need metadata such as updatability, definer, or security type. Cross-schema audits are easiest via `information_schema.VIEWS` without a `TABLE_SCHEMA` filter.
