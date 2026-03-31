# How to Use trim(), ltrim(), rtrim() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, String Functions, trim, ltrim, rtrim, Data Cleaning

Description: Learn how to use trim(), ltrim(), and rtrim() in ClickHouse to remove whitespace and custom characters from string values for data cleaning.

---

## String Trimming Functions Overview

ClickHouse provides several functions for removing characters from the beginning and end of strings:

| Function | Removes From |
|----------|-------------|
| `trim(s)` | Both ends (whitespace) |
| `ltrim(s)` | Left (leading) end (whitespace) |
| `rtrim(s)` | Right (trailing) end (whitespace) |
| `trimLeft(s)` | Left end (whitespace) - alias for ltrim |
| `trimRight(s)` | Right end (whitespace) - alias for rtrim |
| `trimBoth(s)` | Both ends (whitespace) - alias for trim |
| `trim(LEADING chars FROM s)` | Left end with custom chars |
| `trim(TRAILING chars FROM s)` | Right end with custom chars |
| `trim(BOTH chars FROM s)` | Both ends with custom chars |

## Basic Whitespace Trimming

```sql
-- Remove whitespace from both ends
SELECT trim('  hello world  ');     -- 'hello world'
SELECT ltrim('  hello world  ');    -- 'hello world  '
SELECT rtrim('  hello world  ');    -- '  hello world'
SELECT trimBoth('  hello world  '); -- 'hello world' (same as trim)

-- Practical use: clean imported data
CREATE TABLE imported_data (
    id UInt64,
    name String,
    city String,
    email String
) ENGINE = MergeTree()
ORDER BY id;

SELECT
    id,
    trim(name) AS name,
    trim(city) AS city,
    lower(trim(email)) AS email
FROM imported_data
WHERE trim(name) != '';
```

## Trimming Custom Characters

```sql
-- Remove specific characters with SQL TRIM syntax
SELECT trim(BOTH '/' FROM '/api/endpoint/');   -- 'api/endpoint'
SELECT trim(LEADING '0' FROM '00042');          -- '42'
SELECT trim(TRAILING '!' FROM 'Hello!!!');      -- 'Hello'

-- Remove multiple characters (any character in the set)
SELECT trim(BOTH '-_.' FROM '---hello_world...');  -- 'hello_world'
```

## Data Cleaning Use Cases

```sql
-- Clean product codes with leading/trailing dashes
SELECT
    product_id,
    trim(BOTH '-' FROM product_code) AS clean_code,
    trim(name) AS clean_name
FROM products
WHERE product_code LIKE '-%' OR product_code LIKE '%-';

-- Normalize phone numbers - remove spaces and dashes
SELECT
    user_id,
    trim(BOTH ' -()' FROM phone) AS normalized_phone
FROM users
WHERE phone IS NOT NULL;

-- Clean URL paths
SELECT
    trim(BOTH '/' FROM url_path) AS clean_path
FROM web_requests;
```

## Combining trim with Other String Functions

```sql
-- Chain multiple string operations
SELECT
    id,
    -- Clean, lowercase, replace spaces with hyphens (slug)
    replaceAll(lower(trim(name)), ' ', '-') AS slug
FROM articles;

-- Validate after trimming
SELECT
    id,
    trim(email) AS email,
    position(trim(email), '@') > 0 AS has_at_sign
FROM user_registrations;

-- Split trimmed string
SELECT
    id,
    splitByChar(',', trim(tags)) AS tag_array
FROM products;
```

## Detecting Strings That Need Trimming

```sql
-- Find rows with leading or trailing whitespace
SELECT id, name, length(name) AS raw_len, length(trim(name)) AS trimmed_len
FROM users
WHERE length(name) != length(trim(name));

-- Count by whitespace type
SELECT
    countIf(startsWith(name, ' ') OR startsWith(name, char(9))) AS leading_whitespace,
    countIf(endsWith(name, ' ') OR endsWith(name, char(9))) AS trailing_whitespace,
    countIf(length(name) != length(trim(name))) AS any_whitespace
FROM users;
```

## Trim in Materialized Views for ETL

```sql
-- Create cleaned table from raw ingestion
CREATE TABLE raw_customers (
    raw_name String,
    raw_email String,
    raw_phone String,
    ingested_at DateTime DEFAULT now()
) ENGINE = MergeTree()
ORDER BY ingested_at;

CREATE TABLE clean_customers (
    name String,
    email String,
    phone String,
    ingested_at DateTime
) ENGINE = MergeTree()
ORDER BY ingested_at;

CREATE MATERIALIZED VIEW clean_customers_mv TO clean_customers AS
SELECT
    trim(raw_name) AS name,
    lower(trim(raw_email)) AS email,
    trim(BOTH ' -()+' FROM raw_phone) AS phone,
    ingested_at
FROM raw_customers
WHERE length(trim(raw_name)) > 0;  -- skip empty names after trimming
```

## Practical Example: Tag Normalization

```sql
CREATE TABLE article_tags (
    article_id UInt64,
    tag_raw String
) ENGINE = MergeTree()
ORDER BY article_id;

INSERT INTO article_tags VALUES
(1, '  clickhouse  '),
(1, 'analytics '),
(2, ' database'),
(2, '  sql  '),
(3, '#performance');

-- Normalize tags: trim whitespace and # characters
SELECT
    article_id,
    trim(BOTH ' #' FROM tag_raw) AS normalized_tag,
    lower(trim(BOTH ' #' FROM tag_raw)) AS canonical_tag
FROM article_tags
ORDER BY article_id, canonical_tag;
```

## Summary

ClickHouse's `trim()`, `ltrim()`, and `rtrim()` functions remove whitespace from both ends, leading end, and trailing end of strings respectively. The full SQL syntax `trim(LEADING|TRAILING|BOTH 'chars' FROM string)` enables trimming specific character sets. These functions are essential for data cleaning during ETL, normalizing imported data, and preparing strings for comparison or indexing. Combine with `lower()`, `replaceAll()`, and `splitByChar()` for comprehensive string normalization pipelines.
