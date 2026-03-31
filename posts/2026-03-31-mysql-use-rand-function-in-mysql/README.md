# How to Use RAND() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Rand Function, Random Numbers, Math Functions, Sql

Description: Learn how to use the RAND() function in MySQL to generate random floating-point numbers and use them for random row selection, shuffling, and sampling.

---

## Introduction

`RAND()` is a MySQL function that returns a random floating-point value between 0 (inclusive) and 1 (exclusive). It is used for random row selection, shuffling results, generating test data, and random sampling. When called with a seed value, it produces a reproducible sequence.

## Basic Syntax

```sql
RAND()          -- Random value each call
RAND(seed)      -- Reproducible sequence with fixed seed
```

## Basic Examples

```sql
SELECT RAND();          -- Returns something like 0.7452913648456
SELECT RAND();          -- Returns a different value each time
SELECT RAND(42);        -- Returns: 0.6555866465490187 (reproducible)
SELECT RAND(42);        -- Returns the same value with same seed
```

## Generating Random Numbers in a Range

To get a random integer between min and max (inclusive):

```sql
-- Random integer between 1 and 100
SELECT FLOOR(RAND() * 100) + 1 AS random_num;

-- Random integer between min_val and max_val
SELECT FLOOR(RAND() * (max_val - min_val + 1)) + min_val AS random_num;

-- Example: random number between 50 and 100
SELECT FLOOR(RAND() * 51) + 50 AS random_num;
```

## Random Float in a Range

```sql
-- Random float between 1.0 and 10.0
SELECT RAND() * 9 + 1 AS random_float;
```

## Selecting Random Rows

Shuffle and return the top N rows randomly:

```sql
SELECT id, name, email
FROM customers
ORDER BY RAND()
LIMIT 10;
```

This returns 10 random customers. Note: `ORDER BY RAND()` is a full table scan and sort, which is slow on large tables.

## Faster Random Sampling for Large Tables

For large tables, a more efficient approach uses a random offset or probabilistic WHERE:

```sql
-- Probabilistic sampling: select ~1% of rows
SELECT id, name
FROM large_table
WHERE RAND() < 0.01;

-- More rows: select ~10%
SELECT id, name
FROM large_table
WHERE RAND() < 0.10;
```

The result count varies, but performance is much better than ORDER BY RAND().

## Random Row Using MAX(id)

```sql
-- Efficient single random row (assumes contiguous IDs)
SELECT *
FROM products
WHERE id >= FLOOR(RAND() * (SELECT MAX(id) FROM products)) + 1
LIMIT 1;
```

## Using RAND() in INSERT for Test Data

```sql
-- Insert random test records
INSERT INTO test_scores (student_id, score)
SELECT
  id,
  FLOOR(RAND() * 41) + 60  -- Random score between 60 and 100
FROM students;
```

## Random Values for Simulation

```sql
-- Simulate dice rolls (1-6)
SELECT FLOOR(RAND() * 6) + 1 AS dice_roll
FROM (SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5) AS rolls;
```

## Seeded RAND() for Reproducible Results

When the same seed is used, RAND() returns the same sequence - useful for testing:

```sql
-- First call with seed 100
SELECT RAND(100);  -- Always returns 0.3280511885...

-- Second call with seed 100 in same query uses same seed differently
SELECT RAND(100), RAND(100);
-- Both return 0.3280511885... (same seed = same value per call)
```

## Using RAND() with CASE for Random Assignment

```sql
-- Randomly assign users to A/B test groups
SELECT
  id,
  CASE WHEN RAND() < 0.5 THEN 'A' ELSE 'B' END AS test_group
FROM users;
```

## Shuffling an Existing Dataset

```sql
-- Create a shuffled copy of a playlist
INSERT INTO shuffled_playlist (track_id, position, playlist_id)
SELECT
  track_id,
  ROW_NUMBER() OVER (ORDER BY RAND()) AS position,
  playlist_id
FROM playlist_tracks
WHERE playlist_id = 42;
```

## Performance Note

`ORDER BY RAND()` requires MySQL to assign a random value to every row, then sort the entire result - this is O(n log n) and very slow on large tables. For production use with large datasets, prefer probabilistic sampling or the MAX(id) approach.

## Summary

`RAND()` generates a random float between 0 and 1 in MySQL. Use `FLOOR(RAND() * range) + min` to generate integers in a specific range. For random row selection, `ORDER BY RAND() LIMIT N` is simple but slow - use probabilistic WHERE filtering for large tables. Use a seed with `RAND(seed)` for reproducible sequences in testing.
