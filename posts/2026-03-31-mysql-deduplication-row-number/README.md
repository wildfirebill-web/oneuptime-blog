# How to Implement De-Duplication with ROW_NUMBER() in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, ROW_NUMBER, De-Duplication, Window Function, Data Cleaning

Description: Learn how to implement efficient de-duplication in MySQL using ROW_NUMBER() window functions to rank and remove duplicate rows while keeping the canonical record.

---

## Why ROW_NUMBER() for De-Duplication?

`ROW_NUMBER()` assigns a unique sequential number to each row within a partition. When partitioned by the deduplication key and ordered by a preference criterion (e.g., most recent timestamp, lowest ID), rows with `ROW_NUMBER() > 1` are duplicates that can be safely removed.

## Identifying Duplicates with ROW_NUMBER()

Label each row with its rank within its duplicate group:

```sql
SELECT
  id,
  email,
  created_at,
  ROW_NUMBER() OVER (
    PARTITION BY email
    ORDER BY created_at ASC  -- keep the oldest record
  ) AS rn
FROM customers;
```

Rows where `rn > 1` are duplicates.

## Deleting Duplicates in MySQL 8.0+

Use a CTE or derived table with `DELETE ... JOIN`:

```sql
DELETE c
FROM customers c
JOIN (
  SELECT id,
         ROW_NUMBER() OVER (
           PARTITION BY email
           ORDER BY created_at ASC
         ) AS rn
  FROM customers
) ranked ON c.id = ranked.id
WHERE ranked.rn > 1;
```

Note: MySQL does not allow direct `DELETE` from a CTE target, so the `JOIN` approach is required.

## Keeping the Most Recent Record

To keep the newest record (highest ID or latest `updated_at`):

```sql
DELETE c
FROM customers c
JOIN (
  SELECT id,
         ROW_NUMBER() OVER (
           PARTITION BY email
           ORDER BY id DESC  -- keep highest ID
         ) AS rn
  FROM customers
) ranked ON c.id = ranked.id
WHERE ranked.rn > 1;
```

## Multi-Column De-Duplication Key

Deduplicate on a compound key (first name + last name + date of birth):

```sql
DELETE c
FROM customers c
JOIN (
  SELECT id,
         ROW_NUMBER() OVER (
           PARTITION BY first_name, last_name, dob
           ORDER BY id ASC
         ) AS rn
  FROM customers
) ranked ON c.id = ranked.id
WHERE ranked.rn > 1;
```

## De-Duplication into a New Table

For large tables, copying non-duplicates is faster than deleting:

```sql
CREATE TABLE customers_clean AS
SELECT *
FROM (
  SELECT *,
         ROW_NUMBER() OVER (
           PARTITION BY email
           ORDER BY created_at ASC
         ) AS rn
  FROM customers
) ranked
WHERE rn = 1;

-- Remove the rn column
ALTER TABLE customers_clean DROP COLUMN rn;

-- Verify counts
SELECT COUNT(*) FROM customers;
SELECT COUNT(*) FROM customers_clean;

-- Swap tables
RENAME TABLE customers TO customers_backup, customers_clean TO customers;
```

## De-Duplication with Tie-Breaking Logic

When multiple preference criteria exist, use a computed expression in ORDER BY:

```sql
DELETE c
FROM customers c
JOIN (
  SELECT id,
         ROW_NUMBER() OVER (
           PARTITION BY email
           ORDER BY
             (phone IS NOT NULL) DESC,   -- prefer rows with phone
             (address IS NOT NULL) DESC,  -- then rows with address
             created_at ASC              -- then oldest
         ) AS rn
  FROM customers
) ranked ON c.id = ranked.id
WHERE ranked.rn > 1;
```

## Summary

Use `ROW_NUMBER() OVER (PARTITION BY dedup_key ORDER BY preference)` to rank rows within duplicate groups. Delete rows with `rn > 1` using a `DELETE ... JOIN` pattern. For large tables, use a table copy approach to keep the non-duplicate rows. Add a `UNIQUE INDEX` after de-duplication to prevent future duplicates.
