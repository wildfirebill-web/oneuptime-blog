# How to Use TRIM() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, String Function, Database

Description: Learn how TRIM() removes leading and trailing characters from strings in MySQL, how to specify custom characters to remove, and how it differs from LTRIM() and RTRIM().

---

`TRIM()` removes leading characters, trailing characters, or both from a string. By default it removes space characters. You can specify a custom character or substring to remove, and control which side to trim.

## Syntax

```sql
TRIM([{BOTH | LEADING | TRAILING} [remstr] FROM] str)
TRIM([remstr FROM] str)
TRIM(str)
```

- `BOTH` (default) - removes from both ends.
- `LEADING` - removes from the start only.
- `TRAILING` - removes from the end only.
- `remstr` - the character or string to remove (default is a space).
- `str` - the source string.

## Basic usage: remove whitespace

```sql
SELECT TRIM('  Hello World  ');
-- 'Hello World'

SELECT TRIM('   MySQL   ');
-- 'MySQL'
```

## LEADING and TRAILING variants

```sql
SELECT TRIM(LEADING  ' ' FROM '  Hello  ');  -- 'Hello  '
SELECT TRIM(TRAILING ' ' FROM '  Hello  ');  -- '  Hello'
SELECT TRIM(BOTH     ' ' FROM '  Hello  ');  -- 'Hello'
```

## Removing a custom character

```sql
SELECT TRIM('/' FROM '/path/to/resource/');
-- 'path/to/resource'

SELECT TRIM(LEADING  '0' FROM '000123');
-- '123'

SELECT TRIM(TRAILING '.' FROM 'filename.');
-- 'filename'
```

`TRIM` removes only the specified `remstr` character or substring from the edges, not from the middle of the string.

## Removing a multi-character prefix or suffix

```sql
SELECT TRIM(LEADING  'www.' FROM 'www.example.com');
-- 'example.com'

SELECT TRIM(TRAILING '.com' FROM 'example.com');
-- 'example'
```

Note: `TRIM` removes only complete occurrences of `remstr` from the edge. It will not partially match.

## Cleaning user input in a table

```sql
CREATE TABLE form_submissions (
    submission_id INT PRIMARY KEY,
    email         VARCHAR(100),
    first_name    VARCHAR(50),
    last_name     VARCHAR(50)
);

INSERT INTO form_submissions VALUES
(1, '  alice@example.com ', '  Alice ', ' Smith'),
(2, 'bob@example.com',      'Bob',      'Jones  ');

-- Clean whitespace from all fields
SELECT
    submission_id,
    TRIM(email)      AS email,
    TRIM(first_name) AS first_name,
    TRIM(last_name)  AS last_name
FROM form_submissions;
```

## Updating existing data

```sql
UPDATE form_submissions
SET
    email      = TRIM(email),
    first_name = TRIM(first_name),
    last_name  = TRIM(last_name)
WHERE first_name != TRIM(first_name)
   OR last_name  != TRIM(last_name)
   OR email      != TRIM(email);
```

## TRIM in WHERE clause

```sql
-- Find rows where the email still has leading or trailing spaces
SELECT submission_id, email
FROM form_submissions
WHERE email != TRIM(email);
```

## TRIM vs LTRIM vs RTRIM

```sql
SELECT TRIM('  Hello  ');   -- 'Hello'   (both sides, spaces)
SELECT LTRIM('  Hello  ');  -- 'Hello  ' (left only, spaces)
SELECT RTRIM('  Hello  ');  -- '  Hello' (right only, spaces)
```

`LTRIM` and `RTRIM` only remove spaces. `TRIM` can remove custom characters from either or both sides.

## Handling NULL

```sql
SELECT TRIM(NULL);    -- NULL
SELECT TRIM('  ');    -- '' (empty string, not NULL)
```

## Nested TRIM for multiple characters

`TRIM` removes only one type of character at a time. To strip both spaces and dashes:

```sql
SELECT TRIM('-' FROM TRIM(' ' FROM '  --hello--  '));
-- 'hello'
```

## Performance note

`TRIM` on an indexed column in a `WHERE` condition prevents index usage:

```sql
-- Index on email cannot be used
WHERE TRIM(email) = 'alice@example.com';

-- Better: clean data at insert/update time, then query the clean value
WHERE email = 'alice@example.com';
```

## Summary

`TRIM()` removes leading and/or trailing characters from a string, defaulting to spaces. Use `LEADING` or `TRAILING` keywords to trim only one side, and specify a `remstr` argument to remove a custom character or substring. For production use, clean data at write time so that queries on indexed columns remain efficient.
