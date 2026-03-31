# How to Handle NULL Values During Data Import in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, NULL, Import, LOAD DATA INFILE, Data Quality

Description: Learn how to handle NULL values, empty strings, and missing fields correctly when importing data into MySQL using LOAD DATA INFILE and SET clauses.

---

## Introduction

NULL handling is one of the trickier aspects of bulk data import. Different source systems represent missing values in different ways: empty strings, the literal text "NULL", `\N` (MySQL's NULL escape), or simply absent fields. Getting this right prevents data quality issues downstream.

## MySQL's Default NULL Representation

MySQL's `LOAD DATA INFILE` recognizes `\N` as NULL by default. A CSV where the third field is NULL:

```text
1,Alice,\N,active
2,Bob,bob@example.com,inactive
```

```sql
LOAD DATA INFILE '/tmp/users.csv'
INTO TABLE users
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
(id, name, email, status);
```

Row 1 will have `email = NULL` automatically.

## Handling Empty Strings as NULL

When the source file uses empty strings to represent missing values:

```text
1,Alice,,active
2,Bob,bob@example.com,inactive
```

Use a user variable and the `SET` clause to convert empty strings to NULL:

```sql
LOAD DATA INFILE '/tmp/users.csv'
INTO TABLE users
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
(@id, @name, @email, @status)
SET
  id     = NULLIF(@id, ''),
  name   = NULLIF(@name, ''),
  email  = NULLIF(@email, ''),
  status = NULLIF(@status, '');
```

`NULLIF(expr, '')` returns NULL when the value is an empty string, otherwise returns the value.

## Handling Literal "NULL" Text as NULL

When the source contains the string `NULL` (not MySQL's `\N`):

```sql
LOAD DATA INFILE '/tmp/users.csv'
INTO TABLE users
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
(@id, @name, @email, @status)
SET
  id     = NULLIF(@id, 'NULL'),
  name   = NULLIF(@name, 'NULL'),
  email  = NULLIF(@email, 'NULL'),
  status = NULLIF(@status, 'NULL');
```

## Combining Multiple NULL Representations

When a file may use either empty strings or the literal "NULL":

```sql
LOAD DATA INFILE '/tmp/mixed_nulls.csv'
INTO TABLE users
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
(@id, @name, @email, @status)
SET
  id     = @id,
  name   = IF(@name IN ('', 'NULL', 'N/A'), NULL, @name),
  email  = IF(@email IN ('', 'NULL', 'N/A'), NULL, @email),
  status = NULLIF(@status, '');
```

## Providing Default Values for Missing Fields

When a column is missing from the CSV but has a NOT NULL constraint, provide a default:

```sql
LOAD DATA INFILE '/tmp/users_partial.csv'
INTO TABLE users
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
(@id, @name, @email)
SET
  id     = @id,
  name   = @name,
  email  = NULLIF(@email, ''),
  status = 'active',
  created_at = NOW();
```

## Verifying NULL Handling After Import

Check that NULLs were inserted as expected:

```sql
SELECT COUNT(*) AS total_rows FROM users;
SELECT COUNT(*) AS rows_with_null_email FROM users WHERE email IS NULL;
SELECT * FROM users WHERE email IS NULL LIMIT 10;
```

## Summary

`LOAD DATA INFILE` handles MySQL's native `\N` NULL marker automatically, but real-world files frequently use empty strings or literal "NULL" text. Use user variables combined with `NULLIF()` or `IF()` in the `SET` clause to normalize any NULL representation into actual SQL NULLs. Always verify NULL counts after import to confirm the mapping worked as intended.
