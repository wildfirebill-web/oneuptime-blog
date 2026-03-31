# MySQL String Functions Cheat Sheet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, String Function, Query, Cheat Sheet

Description: Quick reference for MySQL string functions including CONCAT, SUBSTRING, REPLACE, TRIM, LENGTH, UPPER, LOWER, REGEXP, and more with practical examples.

---

## Concatenation

```sql
-- CONCAT
SELECT CONCAT('Hello', ' ', 'World');        -- 'Hello World'

-- CONCAT_WS (with separator)
SELECT CONCAT_WS(', ', 'Smith', 'John');     -- 'Smith, John'
```

## Length and Position

```sql
SELECT LENGTH('MySQL');          -- 5 (bytes)
SELECT CHAR_LENGTH('MySQL');     -- 5 (characters, differs for multibyte)

SELECT LOCATE('SQL', 'MySQL');   -- 3
SELECT INSTR('MySQL', 'SQL');    -- 3

SELECT POSITION('SQL' IN 'MySQL');  -- 3
```

## Substring Operations

```sql
SELECT SUBSTRING('Hello World', 7);        -- 'World'
SELECT SUBSTRING('Hello World', 1, 5);     -- 'Hello'
SELECT SUBSTR('Hello World', -5);          -- 'World'
SELECT MID('Hello World', 7, 5);           -- 'World'

SELECT LEFT('Hello World', 5);   -- 'Hello'
SELECT RIGHT('Hello World', 5);  -- 'World'
```

## Case Conversion

```sql
SELECT UPPER('hello');    -- 'HELLO'
SELECT LOWER('HELLO');    -- 'hello'
SELECT UCASE('hello');    -- 'HELLO'  (alias)
SELECT LCASE('HELLO');    -- 'hello'  (alias)
```

## Trimming and Padding

```sql
SELECT TRIM('  hello  ');           -- 'hello'
SELECT LTRIM('  hello');            -- 'hello'
SELECT RTRIM('hello  ');            -- 'hello'
SELECT TRIM(LEADING '0' FROM '007');  -- '7'

SELECT LPAD('42', 6, '0');   -- '000042'
SELECT RPAD('hi', 5, '!');   -- 'hi!!!'
```

## Search and Replace

```sql
SELECT REPLACE('foo bar foo', 'foo', 'baz');  -- 'baz bar baz'

-- REGEXP_REPLACE (MySQL 8.0+)
SELECT REGEXP_REPLACE('abc123def', '[0-9]+', '#');  -- 'abc#def'
```

## Formatting

```sql
SELECT FORMAT(1234567.891, 2);       -- '1,234,567.89'
SELECT REPEAT('ha', 3);              -- 'hahaha'
SELECT REVERSE('MySQL');             -- 'LQSyM'
SELECT SPACE(5);                     -- '     ' (5 spaces)
```

## Regular Expressions

```sql
-- REGEXP / RLIKE
SELECT 'user@example.com' REGEXP '^[a-z]+@[a-z]+\\.[a-z]+$';  -- 1

-- REGEXP_LIKE (MySQL 8.0+)
SELECT REGEXP_LIKE('abc123', '^[a-z]+[0-9]+$');  -- 1

-- REGEXP_SUBSTR
SELECT REGEXP_SUBSTR('phone: 555-1234', '[0-9-]+');  -- '555-1234'

-- REGEXP_INSTR
SELECT REGEXP_INSTR('abc123', '[0-9]+');  -- 4
```

## Conversion and Encoding

```sql
SELECT ASCII('A');           -- 65
SELECT CHAR(65);             -- 'A'
SELECT HEX('MySQL');         -- '4D7953514C'
SELECT UNHEX('4D7953514C');  -- 'MySQL'

SELECT TO_BASE64('hello');                  -- 'aGVsbG8='
SELECT FROM_BASE64('aGVsbG8=');            -- 'hello'
```

## Comparison

```sql
SELECT STRCMP('apple', 'banana');   -- -1 (apple < banana)
SELECT STRCMP('same', 'same');      -- 0

-- Soundex for phonetic comparison
SELECT SOUNDEX('Smith'), SOUNDEX('Smythe');  -- same code
```

## Summary

MySQL's string function library covers concatenation (CONCAT, CONCAT_WS), extraction (SUBSTRING, LEFT, RIGHT), normalization (TRIM, UPPER, LOWER), pattern matching (REGEXP_LIKE, REGEXP_REPLACE), and formatting (LPAD, FORMAT). These functions can eliminate much of the string-processing logic in application code, keeping data transformation closer to the data source.
