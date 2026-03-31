# How to Generate Random Numbers in a Range in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Numeric Function, Random, Database

Description: Learn how to generate random integers within a specific range in MySQL using RAND() and FLOOR(), with examples for test data, sampling, and game logic.

---

## Overview

MySQL's `RAND()` function returns a pseudo-random floating-point number between 0 (inclusive) and 1 (exclusive). To generate random integers within a custom range, combine `RAND()` with `FLOOR()`.

**Formula for a random integer between min and max (inclusive):**

```sql
FLOOR(RAND() * (max - min + 1)) + min
```

## Basic Range Examples

```sql
-- Random integer from 1 to 10
SELECT FLOOR(RAND() * 10) + 1 AS random_1_to_10;

-- Random integer from 0 to 99
SELECT FLOOR(RAND() * 100) AS random_0_to_99;

-- Random integer from 50 to 100
SELECT FLOOR(RAND() * 51) + 50 AS random_50_to_100;

-- Random integer from -10 to 10
SELECT FLOOR(RAND() * 21) - 10 AS random_neg10_to_10;
```

## Generating Multiple Random Values at Once

Use a derived table or CTE to generate several random numbers in one query:

```sql
SELECT
  FLOOR(RAND() * 100) + 1 AS dice_1,
  FLOOR(RAND() * 100) + 1 AS dice_2,
  FLOOR(RAND() * 100) + 1 AS dice_3;
```

Note that each call to `RAND()` in the same row produces a different random value.

## Inserting Test Data with Random Ranges

When seeding a database with test records, use random ranges in `INSERT` statements:

```sql
INSERT INTO products (name, price, stock_quantity)
SELECT
  CONCAT('Product ', n),
  ROUND(RAND() * 99 + 1, 2),          -- price: $1.00 to $100.00
  FLOOR(RAND() * 500) + 1              -- stock: 1 to 500
FROM (
  SELECT a.N + b.N * 10 + 1 AS n
  FROM
    (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) a,
    (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) b
) numbers;
```

## Simulating Dice Rolls

```sql
-- Roll a six-sided die
SELECT FLOOR(RAND() * 6) + 1 AS d6;

-- Roll two dice
SELECT
  FLOOR(RAND() * 6) + 1 AS die1,
  FLOOR(RAND() * 6) + 1 AS die2,
  (FLOOR(RAND() * 6) + 1) + (FLOOR(RAND() * 6) + 1) AS total;
```

## Assigning Users to A/B Test Groups

```sql
UPDATE users
SET ab_group = FLOOR(RAND() * 2)  -- 0 = control, 1 = treatment
WHERE ab_group IS NULL;
```

For three groups (0, 1, 2):

```sql
UPDATE users
SET ab_group = FLOOR(RAND() * 3)
WHERE ab_group IS NULL;
```

## Reproducible Random Sequences with Seeds

Pass a seed to `RAND(seed)` for repeatable results. The same seed always produces the same sequence:

```sql
SELECT FLOOR(RAND(42) * 100) + 1 AS seeded_random;
-- Always returns the same value for seed 42
```

Useful for testing and debugging.

## Generating Random Dates in a Range

Use random numbers to generate random dates between two bounds:

```sql
SELECT
  DATE_ADD('2024-01-01', INTERVAL FLOOR(RAND() * 365) DAY) AS random_date_2024;
```

## Understanding the Formula

The general formula `FLOOR(RAND() * (max - min + 1)) + min` works as follows:
- `RAND()` produces a float in `[0, 1)`
- Multiply by `(max - min + 1)` to scale the range
- `FLOOR()` rounds down to an integer from 0 to `(max - min)`
- Add `min` to shift into the desired range

## Summary

To generate random integers within a specific range in MySQL, use `FLOOR(RAND() * (max - min + 1)) + min`. This pattern is useful for generating test data, simulating random events, running A/B test assignments, and any scenario requiring controlled randomness. Use `RAND(seed)` when you need reproducible results across query runs.
