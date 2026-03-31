# How to Write Basic SELECT Queries in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, SELECT, Query

Description: Learn the fundamentals of SELECT queries in ClickHouse, from column aliases and expressions to system tables and wildcard selects.

---

ClickHouse uses a SQL dialect that is largely compatible with standard SQL while adding its own extensions and optimizations. Understanding the basics of SELECT is the foundation for working effectively with ClickHouse, whether you are querying application logs, metrics, or analytical datasets. This post walks through the core SELECT syntax with practical examples you can run immediately.

## Basic SELECT Syntax

The simplest form of a SELECT statement retrieves all columns from a table using the wildcard `*`.

```sql
SELECT *
FROM system.databases;
```

To select specific columns, list them by name separated by commas.

```sql
SELECT
    name,
    engine,
    data_path
FROM system.databases;
```

### Column Aliases

Use `AS` to assign an alias to a column or expression. Aliases improve readability and can be referenced in `ORDER BY` and `GROUP BY` clauses.

```sql
SELECT
    name AS database_name,
    engine AS storage_engine
FROM system.databases;
```

## Expressions in SELECT

ClickHouse supports arithmetic, string, and function expressions directly in the SELECT list.

```sql
SELECT
    name,
    total_rows,
    total_bytes,
    round(total_bytes / 1048576, 2) AS size_mb
FROM system.tables
WHERE database = 'default'
ORDER BY size_mb DESC;
```

### String Expressions

```sql
SELECT
    concat(database, '.', name) AS full_table_name,
    engine,
    total_rows
FROM system.tables
WHERE total_rows > 0;
```

### Conditional Expressions

Use `if()` or `CASE` for conditional logic in column expressions.

```sql
SELECT
    name,
    total_rows,
    if(total_rows > 1000000, 'large', 'small') AS table_size_class
FROM system.tables
WHERE database = 'default';
```

```sql
SELECT
    name,
    engine,
    CASE
        WHEN engine LIKE 'MergeTree%' THEN 'MergeTree family'
        WHEN engine = 'Log'          THEN 'Log family'
        ELSE 'Other'
    END AS engine_family
FROM system.tables;
```

## Querying SYSTEM Tables

ClickHouse exposes metadata through the `system` database. These tables are useful for exploring your cluster's state without any test data.

```sql
-- List all tables in a database with their engines
SELECT
    name,
    engine,
    total_rows,
    formatReadableSize(total_bytes) AS size
FROM system.tables
WHERE database = 'default';
```

```sql
-- List running processes
SELECT
    query_id,
    user,
    elapsed,
    query
FROM system.processes
ORDER BY elapsed DESC;
```

```sql
-- Check current settings
SELECT
    name,
    value,
    description
FROM system.settings
WHERE name LIKE 'max_%'
LIMIT 20;
```

## Literal Values and Constants

You can SELECT literal values and constants without a FROM clause.

```sql
SELECT
    1 + 1 AS two,
    'hello' AS greeting,
    now() AS current_time,
    version() AS ch_version;
```

## Selecting from a Sample Table

Here is a practical example creating a table and querying it with various SELECT forms.

```sql
-- Create a sample events table
CREATE TABLE events
(
    event_id   UInt64,
    event_type String,
    user_id    UInt32,
    value      Float64,
    created_at DateTime
)
ENGINE = MergeTree()
ORDER BY (event_type, created_at);

-- Insert sample data
INSERT INTO events VALUES
    (1, 'click', 101, 1.5, now()),
    (2, 'view',  102, 0.0, now()),
    (3, 'click', 101, 2.3, now()),
    (4, 'buy',   103, 49.99, now());

-- Select all columns
SELECT * FROM events;

-- Select specific columns with an expression
SELECT
    event_id,
    event_type,
    user_id,
    round(value, 1) AS rounded_value,
    toDate(created_at) AS event_date
FROM events;
```

## Summary

ClickHouse SELECT queries follow standard SQL conventions with powerful extensions such as inline expressions, function calls, and alias reuse in sorting and grouping. The `system` database provides a rich set of metadata tables that let you explore the cluster without setting up test data. Mastering these basics gives you a solid foundation to build more complex analytical queries.
