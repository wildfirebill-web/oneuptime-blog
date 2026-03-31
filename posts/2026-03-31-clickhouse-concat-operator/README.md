# How to Use concat() and the || Operator in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, String Function, SQL, concat

Description: concat() joins multiple strings in ClickHouse; the || operator is an alias. Learn to build full names, URLs, and log messages from columns with practical examples.

---

Concatenating strings is a fundamental SQL operation. ClickHouse provides the `concat()` function as the primary tool for joining strings together, the `||` operator as a readable alternative, and `concatAssumeInjective()` for a performance-optimized variant used in aggregation scenarios. All three join two or more string values into a single result without any separator, unless you add one explicitly.

## concat() Basics

`concat(str1, str2, ...)` accepts any number of arguments and returns them joined together. NULL arguments in ClickHouse string columns are not common (columns default to empty strings), but if you are working with Nullable(String), a single NULL makes the result NULL - use `coalesce()` to guard against that.

```sql
SELECT concat('Hello', ', ', 'World', '!') AS greeting;
```

```text
greeting
--------------
Hello, World!
```

## The || Operator

The `||` operator is syntactic sugar for `concat()`. It is especially readable when joining exactly two values.

```sql
SELECT 'Hello' || ', ' || 'World' || '!' AS greeting;
```

```text
greeting
--------------
Hello, World!
```

Both forms produce identical results. Use whichever reads more clearly in context; `concat()` is preferable when joining many pieces, and `||` is concise for two-part joins.

## Building Full Names from Columns

A common use case is constructing a display name from first and last name columns.

```sql
CREATE TABLE employees
(
    employee_id  UInt32,
    first_name   String,
    last_name    String,
    department   String,
    email_prefix String
)
ENGINE = MergeTree()
ORDER BY employee_id;

INSERT INTO employees VALUES
    (1, 'Alice',   'Chen',    'Engineering', 'achen'),
    (2, 'Bob',     'Nguyen',  'Marketing',   'bnguyen'),
    (3, 'Claire',  'Dubois',  'Engineering', 'cdubois'),
    (4, 'Daniel',  'Okafor',  'Sales',       'dokafor'),
    (5, 'Fatima',  'Al-Rashid','Product',    'falrashid');
```

```sql
SELECT
    employee_id,
    concat(first_name, ' ', last_name) AS full_name,
    department
FROM employees
ORDER BY employee_id;
```

Adding a literal space between first and last name produces a properly formatted full name.

## Building Email Addresses

Combine column values with a literal domain to construct email addresses.

```sql
SELECT
    employee_id,
    concat(first_name, ' ', last_name)      AS full_name,
    concat(email_prefix, '@company.com')    AS work_email
FROM employees
ORDER BY employee_id;
```

Using the `||` operator for the email construction reads cleanly:

```sql
SELECT
    employee_id,
    first_name || ' ' || last_name       AS full_name,
    email_prefix || '@company.com'       AS work_email
FROM employees
ORDER BY employee_id;
```

## Building URL Paths

Concatenation is useful for constructing URLs from a base path and dynamic identifiers.

```sql
SELECT
    employee_id,
    concat(
        'https://hr.company.com/employees/',
        toString(employee_id),
        '/profile'
    ) AS profile_url
FROM employees
ORDER BY employee_id;
```

Note that `toString()` converts the UInt32 `employee_id` to a string before concatenation. ClickHouse's `concat()` requires all arguments to be strings or implicitly castable.

## Constructing Log Messages

Log pipelines often need to assemble structured log lines from multiple event columns.

```sql
CREATE TABLE app_events
(
    event_time DateTime,
    level      String,
    service    String,
    message    String
)
ENGINE = MergeTree()
ORDER BY event_time;

INSERT INTO app_events VALUES
    ('2024-03-15 10:23:45', 'ERROR', 'auth-service',   'Token validation failed'),
    ('2024-03-15 10:23:47', 'INFO',  'auth-service',   'User login successful'),
    ('2024-03-15 10:24:01', 'WARN',  'payment-service','Retry attempt 2 of 3');
```

```sql
SELECT
    concat(
        '[', toString(event_time), '] ',
        '[', level, '] ',
        service, ': ',
        message
    ) AS log_line
FROM app_events
ORDER BY event_time;
```

```text
log_line
-----------------------------------------------------------
[2024-03-15 10:23:45] [ERROR] auth-service: Token validation failed
[2024-03-15 10:23:47] [INFO]  auth-service: User login successful
[2024-03-15 10:24:01] [WARN]  payment-service: Retry attempt 2 of 3
```

## concatAssumeInjective() for Aggregation Optimization

`concatAssumeInjective(str1, str2, ...)` is a variant that tells the ClickHouse query optimizer the mapping from input to output is injective (no two different input combinations produce the same output). This allows certain aggregate queries to optimize by not grouping on the concatenated result but instead on its components.

Use it in GROUP BY scenarios where you are certain the combined key uniquely identifies each group.

```sql
SELECT
    concatAssumeInjective(department, '-', toString(employee_id)) AS dept_key,
    count() AS event_count
FROM app_events
-- hypothetical join scenario
-- this function shines when ClickHouse rewrites aggregations internally
GROUP BY dept_key
ORDER BY dept_key;
```

For most application queries, `concat()` and `||` are sufficient. Reserve `concatAssumeInjective()` for cases where ClickHouse's query planner can make use of the injectivity hint.

## Handling Nullable Strings

If a column is `Nullable(String)`, a NULL value in any argument makes the entire `concat()` result NULL. Use `coalesce()` or `ifNull()` to provide a fallback.

```sql
SELECT
    employee_id,
    concat(
        coalesce(first_name, ''),
        ' ',
        coalesce(last_name, 'Unknown')
    ) AS safe_full_name
FROM employees
ORDER BY employee_id;
```

## Summary

`concat()` and `||` are interchangeable ways to join strings in ClickHouse. Both accept any number of arguments and return their values joined with no separator, unless you include separator strings explicitly. Common patterns include building display names, email addresses, URL paths, and structured log lines from individual column values. Use `toString()` to convert numeric or date columns before concatenation, and guard against NULL with `coalesce()` or `ifNull()` when dealing with Nullable columns.
