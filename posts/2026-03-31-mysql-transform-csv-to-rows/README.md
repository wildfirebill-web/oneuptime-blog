# How to Transform Comma-Separated Values into Rows in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, String, JSON, Normalization, Query

Description: Learn how to split comma-separated values stored in a single MySQL column into individual rows using JSON_TABLE, recursive CTEs, and helper tables.

---

## The Anti-Pattern: Storing CSV in a Column

Storing comma-separated values in a single column - like `"tag1,tag2,tag3"` - violates first normal form and makes querying individual values difficult. MySQL offers several techniques to explode these values into rows.

## Using JSON_TABLE (MySQL 8.0+)

The cleanest approach is `JSON_TABLE`, which can parse a JSON array. First convert the CSV to a JSON array, then use `JSON_TABLE`:

```sql
SELECT
  p.id,
  p.name,
  jt.tag
FROM products p
CROSS JOIN JSON_TABLE(
  CONCAT('["', REPLACE(p.tags, ',', '","'), '"]'),
  '$[*]' COLUMNS (tag VARCHAR(100) PATH '$')
) AS jt
WHERE p.tags IS NOT NULL AND p.tags != '';
```

This converts `"tag1,tag2,tag3"` into a JSON array `["tag1","tag2","tag3"]` and unpacks it into rows.

## Using a Numbers Table

A numbers table provides a row-based iterator to split by position:

```sql
-- Create a numbers helper table
CREATE TABLE numbers (n INT PRIMARY KEY);
INSERT INTO numbers (n)
SELECT a.N + b.N * 10 + c.N * 100 + 1
FROM
  (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4
   UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) a,
  (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4
   UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) b,
  (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3) c;

-- Split CSV into rows using the numbers table
SELECT
  p.id,
  p.name,
  TRIM(SUBSTRING_INDEX(SUBSTRING_INDEX(p.tags, ',', n.n), ',', -1)) AS tag
FROM products p
JOIN numbers n
  ON n.n <= 1 + LENGTH(p.tags) - LENGTH(REPLACE(p.tags, ',', ''))
WHERE p.tags IS NOT NULL;
```

## Using a Recursive CTE (MySQL 8.0+)

Recursive CTEs offer a self-contained approach without helper tables:

```sql
WITH RECURSIVE split AS (
  SELECT
    id,
    tags,
    TRIM(SUBSTRING_INDEX(tags, ',', 1)) AS tag,
    IF(LOCATE(',', tags) > 0, SUBSTRING(tags, LOCATE(',', tags) + 1), NULL) AS remainder
  FROM products
  WHERE tags IS NOT NULL

  UNION ALL

  SELECT
    id,
    tags,
    TRIM(SUBSTRING_INDEX(remainder, ',', 1)),
    IF(LOCATE(',', remainder) > 0, SUBSTRING(remainder, LOCATE(',', remainder) + 1), NULL)
  FROM split
  WHERE remainder IS NOT NULL
)
SELECT id, tags, tag FROM split ORDER BY id;
```

## Inserting Exploded Values into a Proper Table

After splitting, insert the normalized data into a junction table:

```sql
CREATE TABLE product_tags (
  product_id BIGINT NOT NULL,
  tag        VARCHAR(100) NOT NULL,
  PRIMARY KEY (product_id, tag)
) ENGINE=InnoDB;

INSERT INTO product_tags (product_id, tag)
SELECT p.id, jt.tag
FROM products p
CROSS JOIN JSON_TABLE(
  CONCAT('["', REPLACE(p.tags, ',', '","'), '"]'),
  '$[*]' COLUMNS (tag VARCHAR(100) PATH '$')
) AS jt
WHERE p.tags IS NOT NULL
ON DUPLICATE KEY UPDATE tag = VALUES(tag);
```

## Summary

Split comma-separated values into MySQL rows using `JSON_TABLE` (cleanest, MySQL 8.0+), a numbers helper table (widest compatibility), or a recursive CTE (self-contained, MySQL 8.0+). After exploding the data, store it in a normalized junction table and drop the CSV column to complete the normalization.
