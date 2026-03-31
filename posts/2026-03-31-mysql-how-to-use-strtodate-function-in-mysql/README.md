# How to Use STR_TO_DATE() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Str To Date, Date Parsing, Date Functions, Sql

Description: Learn how to use MySQL's STR_TO_DATE() function to parse date and time strings with custom format specifiers into proper DATE or DATETIME values.

---

## Overview

`STR_TO_DATE()` is the MySQL function for converting a string into a `DATE`, `TIME`, or `DATETIME` value using a format mask. It is the inverse of `DATE_FORMAT()` and is essential for importing data with non-standard date strings or for filtering on formatted date columns.

## Basic Syntax

```sql
STR_TO_DATE(str, format)
```

The format uses the same specifiers as `DATE_FORMAT()`. Returns a `DATE`, `TIME`, or `DATETIME` depending on the format specifiers used.

## Common Format Specifiers

```text
%Y - 4-digit year (2024)
%y - 2-digit year (24)
%m - Month number, zero-padded (01-12)
%c - Month number, no padding (1-12)
%d - Day of month, zero-padded (01-31)
%e - Day of month, no padding (1-31)
%H - Hour (00-23)
%h - Hour (01-12)
%i - Minutes (00-59)
%s - Seconds (00-59)
%p - AM or PM
%M - Full month name (January)
%b - Abbreviated month name (Jan)
%W - Full weekday name (Monday)
```

## Basic Examples

```sql
-- Parse a US-style date
SELECT STR_TO_DATE('12/25/2024', '%m/%d/%Y');  -- 2024-12-25

-- Parse a European-style date
SELECT STR_TO_DATE('25-12-2024', '%d-%m-%Y');  -- 2024-12-25

-- Parse a date and time
SELECT STR_TO_DATE('2024-06-15 14:30:00', '%Y-%m-%d %H:%i:%s');  -- 2024-06-15 14:30:00

-- Parse 12-hour time
SELECT STR_TO_DATE('06/15/2024 02:30 PM', '%m/%d/%Y %h:%i %p');  -- 2024-06-15 14:30:00

-- Parse with full month name
SELECT STR_TO_DATE('June 15, 2024', '%M %d, %Y');  -- 2024-06-15
```

## Importing Data with Mixed Date Formats

```sql
-- Insert data with a non-standard date string
INSERT INTO events (event_name, event_date)
VALUES ('Conference', STR_TO_DATE('15/06/2024', '%d/%m/%Y'));

-- Bulk update to convert string dates to proper DATE type
UPDATE legacy_data
SET parsed_date = STR_TO_DATE(raw_date_string, '%d %b %Y')
WHERE raw_date_string REGEXP '^[0-9]{1,2} [A-Za-z]{3} [0-9]{4}$';
```

## Comparing STR_TO_DATE() with Date Literals

```sql
-- Filter records using a formatted string date
SELECT * FROM orders
WHERE order_date = STR_TO_DATE('2024-06-15', '%Y-%m-%d');

-- Range query with string-to-date conversion
SELECT * FROM logs
WHERE log_time BETWEEN
  STR_TO_DATE('2024-01-01 00:00:00', '%Y-%m-%d %H:%i:%s') AND
  STR_TO_DATE('2024-03-31 23:59:59', '%Y-%m-%d %H:%i:%s');
```

## Handling Invalid Dates

```sql
-- Returns NULL for invalid dates
SELECT STR_TO_DATE('2024-02-30', '%Y-%m-%d');  -- NULL

-- Check SQL mode - strict mode raises an error instead of returning NULL
SELECT @@sql_mode;

-- Use IFNULL to handle NULL results
SELECT IFNULL(STR_TO_DATE(user_input, '%d/%m/%Y'), 'Invalid date') AS result;
```

## Practical Example: Cleaning Imported CSV Data

```sql
CREATE TABLE raw_imports (
  id INT AUTO_INCREMENT PRIMARY KEY,
  raw_date VARCHAR(50),
  amount DECIMAL(10,2)
);

INSERT INTO raw_imports (raw_date, amount) VALUES
('01/15/2024', 100.00),
('02/28/2024', 250.50),
('invalid', 75.00);

-- Create a clean table from the imports
SELECT
  id,
  STR_TO_DATE(raw_date, '%m/%d/%Y') AS clean_date,
  amount
FROM raw_imports
WHERE STR_TO_DATE(raw_date, '%m/%d/%Y') IS NOT NULL;
```

## STR_TO_DATE() vs CAST() and CONVERT()

```sql
-- CAST expects standard format (YYYY-MM-DD)
SELECT CAST('2024-06-15' AS DATE);  -- works
SELECT CAST('15/06/2024' AS DATE);  -- NULL or error

-- STR_TO_DATE handles custom formats
SELECT STR_TO_DATE('15/06/2024', '%d/%m/%Y');  -- 2024-06-15
```

Use `STR_TO_DATE()` when the input string format is not the MySQL standard `YYYY-MM-DD`.

## Summary

`STR_TO_DATE()` parses date and time strings into proper MySQL date types using format specifiers identical to those in `DATE_FORMAT()`. It is the preferred way to handle non-standard date strings from user input, CSV imports, or legacy systems. Always validate that the format mask exactly matches the input string format, and use `IFNULL` or `WHERE IS NOT NULL` to handle invalid inputs gracefully.
