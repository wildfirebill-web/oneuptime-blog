# How to Use NULLIF() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Nullif, Null Handling, Sql, Functions

Description: Learn how to use the NULLIF() function in MySQL to return NULL when two expressions are equal, preventing division by zero and normalizing data.

---

## Introduction

`NULLIF()` is a MySQL function that compares two expressions and returns NULL if they are equal, or the first expression if they are not equal. It is the inverse of `COALESCE()` - instead of replacing NULL with a value, it converts a value into NULL under a specific condition.

## Syntax

```sql
NULLIF(expression1, expression2)
```

- Returns NULL if `expression1 = expression2`.
- Returns `expression1` if `expression1 != expression2`.

It is equivalent to:

```sql
CASE WHEN expression1 = expression2 THEN NULL ELSE expression1 END
```

## Basic Examples

```sql
SELECT NULLIF(5, 5);
-- Returns: NULL

SELECT NULLIF(5, 10);
-- Returns: 5

SELECT NULLIF('hello', 'hello');
-- Returns: NULL

SELECT NULLIF('hello', 'world');
-- Returns: 'hello'
```

## Preventing Division by Zero

The most common use case for `NULLIF()` is avoiding division by zero errors.

```sql
-- Without NULLIF: causes error if total_visits = 0
SELECT conversions / total_visits AS rate FROM stats;

-- With NULLIF: returns NULL instead of error
SELECT conversions / NULLIF(total_visits, 0) AS rate FROM stats;
```

## Normalizing Empty Strings to NULL

Applications sometimes store empty strings instead of NULL for missing data. Use `NULLIF` to normalize them.

```sql
SELECT
  id,
  NULLIF(phone, '') AS phone,
  NULLIF(address, '') AS address
FROM contacts;
```

## Combining NULLIF with COALESCE

A powerful pattern is using `NULLIF` to convert a sentinel value to NULL, then `COALESCE` to provide a fallback.

```sql
-- Convert 'N/A' to NULL, then fall back to 'Unknown'
SELECT COALESCE(NULLIF(status, 'N/A'), 'Unknown') AS status
FROM records;

-- Convert 0 to NULL, then fall back to the default rate
SELECT COALESCE(NULLIF(custom_rate, 0), default_rate) AS rate
FROM pricing;
```

## NULLIF in Aggregations

Use `NULLIF` to exclude specific values from aggregate calculations. Aggregate functions like `AVG()`, `SUM()`, and `COUNT()` ignore NULLs.

```sql
-- Average score, ignoring scores of -1 (sentinel for "not taken")
SELECT AVG(NULLIF(score, -1)) AS avg_score
FROM exam_results;

-- Count non-empty responses
SELECT COUNT(NULLIF(response, '')) AS valid_responses
FROM survey;
```

## NULLIF in UPDATE

Convert placeholder values to actual NULL in existing data.

```sql
UPDATE employees
SET manager_id = NULLIF(manager_id, 0)
WHERE manager_id = 0;
```

## NULLIF with Dates

```sql
-- Treat '0000-00-00' as NULL (common legacy issue)
SELECT
  name,
  NULLIF(birth_date, '0000-00-00') AS birth_date
FROM users;
```

## NULLIF vs IF

```sql
-- NULLIF is shorter but limited to NULL output
SELECT NULLIF(x, 0) FROM t;

-- IF is more flexible
SELECT IF(x = 0, NULL, x) FROM t;
```

## Type Considerations

Both arguments to `NULLIF` should be the same or compatible data types. MySQL performs implicit type conversion when needed.

```sql
SELECT NULLIF(1, '1');
-- Returns NULL because 1 = 1 after type conversion
```

## Practical Example: Safe Percentage Calculation

```sql
SELECT
  department,
  passed_count,
  total_count,
  ROUND(passed_count / NULLIF(total_count, 0) * 100, 2) AS pass_rate
FROM exam_summary;
```

## Summary

`NULLIF()` converts a specific value into NULL when a condition is met. Its primary use cases are preventing division by zero, normalizing sentinel values like empty strings or zeros to NULL, and working with aggregate functions that skip NULLs. Combine it with `COALESCE()` for powerful NULL-handling patterns.
