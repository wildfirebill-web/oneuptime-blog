# How to Use MOD() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Mod Function, Modulo, Math Functions, Sql

Description: Learn how to use the MOD() function and % operator in MySQL to compute the remainder of division, with examples for odd/even checks and cyclic patterns.

---

## Introduction

The `MOD()` function in MySQL returns the remainder of a division operation - also known as the modulo operation. It has two forms: the function `MOD(N, M)` and the operator `N % M` (or `N MOD M`). Modulo is useful for determining odd/even values, cyclic patterns, distributing rows into groups, and implementing hash-based partitioning.

## Basic Syntax

```sql
MOD(dividend, divisor)
-- OR
dividend % divisor
-- OR
dividend MOD divisor
```

All three forms are equivalent.

## Basic Examples

```sql
SELECT MOD(10, 3);   -- Returns: 1  (10 = 3*3 + 1)
SELECT MOD(15, 5);   -- Returns: 0  (15 = 5*3 + 0, evenly divisible)
SELECT MOD(7, 2);    -- Returns: 1  (7 is odd)
SELECT 10 % 3;       -- Returns: 1  (operator form)
SELECT 17 MOD 5;     -- Returns: 2  (keyword form)
```

## Checking Even and Odd Numbers

```sql
-- Find even-numbered rows
SELECT id, name
FROM employees
WHERE MOD(id, 2) = 0;

-- Find odd-numbered rows
SELECT id, name
FROM employees
WHERE MOD(id, 2) = 1;
-- Or equivalently: WHERE id % 2 != 0
```

## Dividing Rows into Groups

Distribute rows evenly among N buckets (useful for load balancing or sampling):

```sql
-- Divide into 4 groups based on id
SELECT id, name, MOD(id, 4) AS group_number
FROM users
ORDER BY group_number, id;
```

## MOD for Pagination Logic

Calculate which page a row belongs to:

```sql
-- Row 1-10 = page 1, 11-20 = page 2, etc.
SELECT
  id,
  FLOOR((id - 1) / 10) + 1 AS page_number,
  MOD(id - 1, 10) + 1 AS position_on_page
FROM records;
```

## MOD with Decimal Numbers

```sql
SELECT MOD(10.5, 3.2);  -- Returns: 0.9 (10.5 = 3.2*3 + 0.9)
SELECT MOD(5.5, 2.5);   -- Returns: 0.5
```

## MOD with Negative Numbers

```sql
SELECT MOD(-10, 3);   -- Returns: -1  (result has sign of dividend)
SELECT MOD(10, -3);   -- Returns: 1   (result has sign of dividend)
SELECT MOD(-10, -3);  -- Returns: -1
```

The sign of the result follows the dividend (the first argument).

## MOD Zero

```sql
SELECT MOD(10, 0);  -- Returns: NULL (division by zero)
```

## Cyclic Counter

Create a cyclic sequence that wraps around:

```sql
-- Create values 0-6 repeating (for day-of-week cycling)
SELECT id, MOD(id - 1, 7) AS day_slot
FROM schedule_slots;
```

## Selecting Every Nth Row

```sql
-- Select every 5th row (for sampling)
SELECT id, data
FROM large_table
WHERE MOD(id, 5) = 0;
```

## Hash-Based Sharding

Determine which shard a record belongs to:

```sql
-- 8 shards
SELECT
  customer_id,
  MOD(customer_id, 8) AS shard_id
FROM customers;
```

## MOD in UPDATE

Apply logic only to even-ID records:

```sql
UPDATE products
SET featured = 1
WHERE MOD(id, 2) = 0 AND category = 'Electronics';
```

## MOD vs DIV

`MOD` returns the remainder. `DIV` returns the integer quotient.

```sql
SELECT 17 MOD 5;  -- Returns: 2  (remainder)
SELECT 17 DIV 5;  -- Returns: 3  (quotient: how many times 5 fits into 17)
```

Together they describe integer division: `17 = 5 * (17 DIV 5) + (17 MOD 5)` = `5*3 + 2`.

## Summary

`MOD()` computes the remainder after division in MySQL, available as a function `MOD(N, M)`, an operator `N % M`, or a keyword `N MOD M`. It is essential for odd/even checks, cyclic sequences, bucket partitioning, row sampling, and hash-based sharding. Division by zero returns NULL.
