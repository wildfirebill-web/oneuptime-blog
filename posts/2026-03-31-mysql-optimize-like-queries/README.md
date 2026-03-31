# How to Optimize LIKE Queries in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Query, Index, Optimization, Performance

Description: Learn how to optimize MySQL LIKE queries by understanding when indexes apply, using prefix patterns, and leveraging FULLTEXT indexes for substring searches.

---

## How LIKE Uses (or Skips) Indexes

LIKE queries have specific rules about when MySQL can use an index:

- `LIKE 'prefix%'` - Can use a B-Tree index (trailing wildcard only)
- `LIKE '%suffix'` - Cannot use a B-Tree index (leading wildcard)
- `LIKE '%substring%'` - Cannot use a B-Tree index (wildcards on both sides)

This is one of the most common sources of unexpected full table scans.

## Checking with EXPLAIN

```sql
-- Trailing wildcard - index friendly
EXPLAIN SELECT * FROM products WHERE name LIKE 'Apple%';
```

```text
type: range, key: idx_name, rows: 150
```

```sql
-- Leading wildcard - no index
EXPLAIN SELECT * FROM products WHERE name LIKE '%phone%';
```

```text
type: ALL, key: NULL, rows: 50000
```

## Prefix Search with Index (Efficient)

For search patterns where users type the beginning of a value, prefix LIKE works well:

```sql
CREATE INDEX idx_name ON products(name);

-- Efficiently uses the index
SELECT id, name, price FROM products WHERE name LIKE 'Smart%';
```

This works because B-Tree indexes store values in sorted order - MySQL can jump to the first entry starting with "Smart" and scan forward.

## Handling Leading Wildcard Searches

Leading wildcard patterns require a different approach. Options include:

### Option 1: FULLTEXT Index

```sql
ALTER TABLE products ADD FULLTEXT INDEX ft_name (name);

-- Search for any occurrence of the word
SELECT id, name FROM products
WHERE MATCH(name) AGAINST('phone' IN BOOLEAN MODE);

-- Prefix search in boolean mode
SELECT id, name FROM products
WHERE MATCH(name) AGAINST('smart*' IN BOOLEAN MODE);
```

FULLTEXT is designed for word-level searches in text columns.

### Option 2: Reverse Index for Suffix Searches

If users always search by suffix (e.g., file extensions), store the reversed string:

```sql
ALTER TABLE files ADD COLUMN name_reversed VARCHAR(255)
  AS (REVERSE(name)) STORED;
CREATE INDEX idx_name_rev ON files(name_reversed);

-- Search for files ending in .pdf
SELECT id, name FROM files WHERE name_reversed LIKE REVERSE('%.pdf');
-- Becomes: WHERE name_reversed LIKE 'fdp.%'
-- Now a prefix search - can use index!
```

### Option 3: Generated Column for Fixed-Pattern Extraction

```sql
-- If you always search by domain part of an email
ALTER TABLE users ADD COLUMN email_domain VARCHAR(100)
  AS (SUBSTRING_INDEX(email, '@', -1)) STORED;
CREATE INDEX idx_email_domain ON users(email_domain);

SELECT * FROM users WHERE email_domain = 'example.com';
```

## LIKE with Case Sensitivity

LIKE follows the column's collation. For case-insensitive searches on a case-sensitive column:

```sql
-- This may not use the index if collations differ
SELECT * FROM products WHERE name LIKE 'apple%' COLLATE utf8mb4_general_ci;

-- Better: ensure the column collation matches your query pattern
-- Or use a separate lowercased generated column
ALTER TABLE products ADD COLUMN name_lower VARCHAR(255)
  AS (LOWER(name)) STORED;
CREATE INDEX idx_name_lower ON products(name_lower);

SELECT * FROM products WHERE name_lower LIKE 'apple%';
```

## Partial Index with Prefix Length

For long VARCHAR columns, use a prefix index to save space:

```sql
-- Index only the first 20 characters
CREATE INDEX idx_name_prefix ON products(name(20));

-- Only useful for prefix LIKE patterns
SELECT * FROM products WHERE name LIKE 'Samsung Galaxy S%';
```

Note: a prefix index cannot be a covering index and may have lower selectivity than a full-column index.

## Summary

LIKE query optimization in MySQL depends entirely on whether you have a leading wildcard. Trailing-wildcard patterns (`'prefix%'`) use B-Tree indexes efficiently. Leading-wildcard patterns (`'%suffix'`, `'%middle%'`) require alternatives: FULLTEXT indexes for word searches, reversed generated columns for suffix patterns, or stored lowercase columns for case-insensitive prefix searches. Always verify with EXPLAIN that your chosen approach actually uses an index.
