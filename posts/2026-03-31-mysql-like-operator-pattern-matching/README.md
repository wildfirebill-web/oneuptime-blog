# How to Use the LIKE Operator for Pattern Matching in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, LIKE Operator, Pattern Matching

Description: Learn how to use the MySQL LIKE operator with % and _ wildcards to filter text data with partial matches, prefix, suffix, and substring patterns.

---

The `LIKE` operator in MySQL performs pattern matching on string values. It supports two wildcards: `%` matches any sequence of characters, and `_` matches any single character. It is useful for searching names, emails, codes, and other text fields.

## Basic Prefix Match

```sql
-- Find all customers whose last name starts with 'Smith'
SELECT customer_id, first_name, last_name
FROM customers
WHERE last_name LIKE 'Smith%';
```

The `%` at the end matches any characters following "Smith", including no characters. This returns "Smith", "Smithson", "Smithwick", etc.

## Suffix Match

```sql
-- Find all email addresses from gmail.com
SELECT email
FROM users
WHERE email LIKE '%@gmail.com';
```

## Substring Match

```sql
-- Find product names containing 'wireless'
SELECT product_name, price
FROM products
WHERE product_name LIKE '%wireless%';
```

This is the slowest pattern because MySQL cannot use a standard index - it must scan all rows.

## Single Character Wildcard with _

The underscore `_` matches exactly one character:

```sql
-- Find 5-character product codes starting with 'A'
SELECT product_code, product_name
FROM products
WHERE product_code LIKE 'A____';

-- Find phone numbers matching a specific format
SELECT customer_id, phone
FROM customers
WHERE phone LIKE '___-___-____';
```

## NOT LIKE

```sql
-- Exclude test email addresses
SELECT email, name
FROM users
WHERE email NOT LIKE '%test%'
  AND email NOT LIKE '%example%';
```

## Case Sensitivity

`LIKE` is case-insensitive for columns using case-insensitive collations (the default `utf8mb4_0900_ai_ci`):

```sql
-- Both return the same rows with a ci collation
SELECT * FROM users WHERE email LIKE '%GMAIL.COM';
SELECT * FROM users WHERE email LIKE '%gmail.com';

-- Force case-sensitive matching
SELECT * FROM users WHERE email LIKE BINARY '%Gmail.com';
```

## Escaping Wildcard Characters

To match a literal `%` or `_`, use the `ESCAPE` clause:

```sql
-- Find discount codes that contain a literal percent sign
SELECT code, description
FROM discount_codes
WHERE description LIKE '%50\%%' ESCAPE '\';

-- Find column values with an underscore
SELECT * FROM config WHERE key_name LIKE 'setting\_name' ESCAPE '\';
```

## Performance Tips

```sql
-- This uses an index (prefix match)
SELECT * FROM customers WHERE last_name LIKE 'Smith%';

-- This does NOT use an index (leading wildcard forces full scan)
SELECT * FROM customers WHERE last_name LIKE '%smith%';
```

For full-text searches with leading wildcards, consider a `FULLTEXT` index with `MATCH ... AGAINST` instead.

## Summary

The `LIKE` operator filters string columns using `%` (any sequence) and `_` (single character) wildcards. Prefix patterns like `'Smith%'` can use indexes efficiently; patterns with a leading `%` require a full table scan. For case-sensitive matching, use `LIKE BINARY` or a case-sensitive collation. Escape literal `%` and `_` characters with a backslash and the `ESCAPE` clause. For complex text search needs, consider MySQL's `FULLTEXT` indexing.
