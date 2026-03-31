# How to Use CONCAT_WS() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, String Function, Database

Description: Learn how CONCAT_WS() joins strings with a separator in MySQL, automatically skips NULL values, and simplifies building comma-separated lists and formatted strings.

---

`CONCAT_WS()` stands for "Concatenate With Separator." It joins two or more strings using a delimiter you specify as the first argument, and unlike `CONCAT()`, it automatically skips `NULL` arguments.

## Syntax

```sql
CONCAT_WS(separator, str1, str2, str3, ...)
```

- `separator` - the string placed between each argument (required).
- If `separator` is `NULL`, the result is `NULL`.
- `NULL` values in `str1`, `str2`, ... are skipped without leaving a double separator.

## Basic usage

```sql
SELECT CONCAT_WS(', ', 'Alice', 'Smith', 'London');
-- Result: 'Alice, Smith, London'

SELECT CONCAT_WS('-', '2026', '03', '31');
-- Result: '2026-03-31'

SELECT CONCAT_WS(' ', 'Hello', 'World');
-- Result: 'Hello World'
```

## NULL handling: the key advantage over CONCAT()

```sql
-- CONCAT() returns NULL if any argument is NULL
SELECT CONCAT('Alice', NULL, 'Smith');
-- Result: NULL

-- CONCAT_WS() skips NULL arguments
SELECT CONCAT_WS(' ', 'Alice', NULL, 'Smith');
-- Result: 'Alice Smith'
```

This makes `CONCAT_WS()` the right choice when columns may be `NULL`.

## Building full names from nullable columns

```sql
CREATE TABLE contacts (
    contact_id  INT PRIMARY KEY,
    first_name  VARCHAR(50),
    middle_name VARCHAR(50),
    last_name   VARCHAR(50)
);

INSERT INTO contacts VALUES
(1, 'Alice', 'Marie',  'Johnson'),
(2, 'Bob',   NULL,     'Smith'),
(3, 'Carol', 'Ann',    NULL);

SELECT
    contact_id,
    CONCAT_WS(' ', first_name, middle_name, last_name) AS full_name
FROM contacts;
```

Result:

| contact_id | full_name |
|---|---|
| 1 | Alice Marie Johnson |
| 2 | Bob Smith |
| 3 | Carol Ann |

No extra spaces appear because `NULL` arguments are skipped entirely.

## Building addresses

```sql
CREATE TABLE addresses (
    address_id INT PRIMARY KEY,
    street     VARCHAR(100),
    city       VARCHAR(60),
    state      VARCHAR(30),
    zip        VARCHAR(10),
    country    VARCHAR(50)
);

SELECT
    CONCAT_WS(', ', street, city, state, zip, country) AS full_address
FROM addresses;
```

## Generating CSV output

```sql
SELECT
    GROUP_CONCAT(name ORDER BY name SEPARATOR ', ') AS name_list
FROM employees
WHERE department_id = 5;
-- GROUP_CONCAT uses its own separator; CONCAT_WS is for row-level concatenation
```

For row-level CSV-style columns:

```sql
SELECT
    order_id,
    CONCAT_WS(',', product_a, product_b, product_c) AS product_list
FROM bundled_orders;
```

## Using CONCAT_WS with a multi-character separator

```sql
SELECT CONCAT_WS(' | ', 'MySQL', '8.0', 'String Functions');
-- Result: 'MySQL | 8.0 | String Functions'
```

## Combining CONCAT_WS with other functions

```sql
SELECT
    CONCAT_WS(' ', UPPER(first_name), UPPER(last_name)) AS upper_name
FROM contacts;

SELECT
    CONCAT_WS('-', YEAR(NOW()), LPAD(MONTH(NOW()), 2, '0'), LPAD(DAY(NOW()), 2, '0')) AS iso_date;
-- Result: e.g. '2026-03-31'
```

## What happens when separator is NULL

```sql
SELECT CONCAT_WS(NULL, 'Alice', 'Smith');
-- Result: NULL
```

The separator being `NULL` always returns `NULL`, regardless of the other arguments.

## CONCAT_WS vs CONCAT

| Feature | CONCAT_WS | CONCAT |
|---|---|---|
| Separator | Single argument, applied between all | Must repeat manually |
| NULL handling | Skips NULL args | Returns NULL if any arg is NULL |
| Use case | Building delimited strings | Precise concatenation with full control |

## Practical query: build a formatted label

```sql
SELECT
    employee_id,
    CONCAT_WS(' - ', name, COALESCE(department, 'Unassigned'), CONCAT('$', FORMAT(salary, 0))) AS label
FROM employees;
-- Example: 'Alice Johnson - Engineering - $85,000'
```

## Summary

`CONCAT_WS()` is the preferred function for joining multiple string values with a separator in MySQL. Its automatic `NULL`-skipping behaviour makes it safer than `CONCAT()` when columns may be `NULL`. Use it for building full names, formatted addresses, date strings, and any delimited string where some fields may be absent.
