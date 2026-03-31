# How to Use SHOW TABLES in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SHOW TABLES, Database Inspection, Schema Management

Description: Learn how to use SHOW TABLES in MySQL to list tables and views in a database, filter by name patterns, and distinguish tables from views.

---

## What Is SHOW TABLES in MySQL

`SHOW TABLES` lists the tables and views in a MySQL database. It is one of the most frequently used commands for exploring database schemas interactively. You can filter results by name pattern, include another database without switching with `USE`, and use `SHOW FULL TABLES` to distinguish base tables from views.

## Basic Syntax

```sql
SHOW TABLES;
SHOW TABLES FROM database_name;
SHOW TABLES LIKE 'pattern';
SHOW FULL TABLES;
SHOW FULL TABLES FROM database_name LIKE 'pattern';
```

## Listing All Tables in the Current Database

```sql
USE mydb;
SHOW TABLES;
```

```text
+-------------------+
| Tables_in_mydb    |
+-------------------+
| customers         |
| order_items       |
| orders            |
| products          |
+-------------------+
```

## Listing Tables in Another Database

```sql
SHOW TABLES FROM information_schema;
SHOW TABLES FROM mydb;
```

## Filtering with LIKE

Use standard SQL LIKE wildcards (`%` and `_`):

```sql
-- Tables starting with 'order'
SHOW TABLES LIKE 'order%';

-- Tables with 'log' anywhere in the name
SHOW TABLES LIKE '%log%';
```

## SHOW FULL TABLES - Distinguishing Tables from Views

```sql
SHOW FULL TABLES;
```

```text
+-------------------+------------+
| Tables_in_mydb    | Table_type |
+-------------------+------------+
| customers         | BASE TABLE |
| order_items       | BASE TABLE |
| orders            | BASE TABLE |
| order_summary     | VIEW       |
| products          | BASE TABLE |
+-------------------+------------+
```

Filter to show only views:

```sql
SHOW FULL TABLES WHERE Table_type = 'VIEW';
```

Filter to show only base tables:

```sql
SHOW FULL TABLES WHERE Table_type = 'BASE TABLE';
```

## Counting Tables in a Database

```sql
SELECT COUNT(*) AS table_count
FROM information_schema.tables
WHERE table_schema = 'mydb'
  AND table_type = 'BASE TABLE';
```

## Listing Tables with Row Counts and Sizes

For more detail than `SHOW TABLES` provides, query `information_schema.tables`:

```sql
SELECT
    table_name,
    table_rows,
    ROUND((data_length + index_length) / 1024 / 1024, 2) AS size_mb,
    table_type,
    engine,
    create_time
FROM information_schema.tables
WHERE table_schema = 'mydb'
ORDER BY (data_length + index_length) DESC;
```

## Finding Tables Modified Recently

```sql
SELECT table_name, update_time
FROM information_schema.tables
WHERE table_schema = 'mydb'
  AND update_time > DATE_SUB(NOW(), INTERVAL 7 DAY)
ORDER BY update_time DESC;
```

## Scripting with SHOW TABLES Output

In bash, iterate over tables to perform operations:

```bash
TABLES=$(mysql -u root -p"$DB_PASS" -N -e "SHOW TABLES FROM mydb;")
for TABLE in $TABLES; do
    echo "Analyzing: $TABLE"
    mysql -u root -p"$DB_PASS" mydb -e "ANALYZE TABLE $TABLE;"
done
```

## SHOW TABLES vs information_schema.tables

| Use Case | SHOW TABLES | information_schema.tables |
|---|---|---|
| Quick interactive listing | Yes | No (verbose SQL required) |
| Programmatic filtering | Limited (LIKE only) | Full SQL WHERE clauses |
| Table size, row counts | No | Yes |
| Engine, charset details | No | Yes |
| Cross-database queries | Yes (FROM db) | Yes (WHERE table_schema) |

## Summary

`SHOW TABLES` provides a fast, interactive way to list tables and views in a MySQL database. Use `SHOW FULL TABLES` to distinguish base tables from views, combine with `LIKE` for name filtering, and fall back to `information_schema.tables` when you need richer details like row counts, storage sizes, or engine information.
