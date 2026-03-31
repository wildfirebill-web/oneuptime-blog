# How to Use Regular Expressions in MySQL WHERE Clause

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Regular Expression, WHERE Clause, Pattern Matching, Query

Description: Learn how to write MySQL WHERE clauses using regular expressions to filter rows based on complex text patterns.

---

## Using REGEXP in WHERE Clauses

MySQL's `REGEXP` and `RLIKE` operators let you filter rows using regular expression patterns in a `WHERE` clause. This gives you far more expressive pattern matching than `LIKE`.

```sql
SELECT *
FROM table_name
WHERE column_name REGEXP 'pattern';
```

## Filtering by String Format

Find users whose usernames contain only lowercase letters:

```sql
SELECT id, username
FROM users
WHERE username REGEXP '^[a-z]+$';
```

Find orders with a reference number matching a specific format:

```sql
SELECT id, reference
FROM orders
WHERE reference REGEXP '^ORD-[0-9]{6}$';
```

## Filtering Email Domains

Retrieve all users with a corporate email (not Gmail, Yahoo, etc.):

```sql
SELECT id, email
FROM users
WHERE email NOT REGEXP '@(gmail|yahoo|hotmail|outlook)\\.com$';
```

## Filtering IP Addresses

Find log entries from a specific subnet:

```sql
SELECT id, ip_address, event
FROM access_logs
WHERE ip_address REGEXP '^192\\.168\\.1\\.';
```

## Combining REGEXP with Other Conditions

```sql
SELECT id, name, phone
FROM customers
WHERE country = 'US'
  AND phone REGEXP '^\\+1[0-9]{10}$';
```

## Extracting Matches with REGEXP_SUBSTR

MySQL 8.0 added `REGEXP_SUBSTR()` for extracting the matched portion:

```sql
SELECT
  id,
  REGEXP_SUBSTR(description, '[A-Z]{2}-[0-9]+') AS extracted_code
FROM tickets
WHERE description REGEXP '[A-Z]{2}-[0-9]+';
```

## Replacing Patterns with REGEXP_REPLACE

Sanitize data directly in a query or update:

```sql
UPDATE users
SET phone = REGEXP_REPLACE(phone, '[^0-9]', '')
WHERE phone REGEXP '[^0-9+]';
```

## Performance Note

`REGEXP` in a `WHERE` clause cannot use a regular B-tree index and will scan the table or use a full-index scan. For large tables with frequent pattern queries, consider:

- Adding a full-text index for text search use cases
- Storing a normalized version of the column that can use a regular index
- Using generated columns to precompute the filtered value

```sql
EXPLAIN SELECT * FROM users WHERE email REGEXP '@company\\.com$';
-- type: ALL (full table scan) - expected for REGEXP
```

## Summary

Regular expressions in MySQL `WHERE` clauses provide flexible pattern filtering using `REGEXP` or `RLIKE`. MySQL 8.0 also provides `REGEXP_SUBSTR()` and `REGEXP_REPLACE()` for richer manipulation. Because `REGEXP` cannot use standard indexes, use it selectively on small tables or combine it with indexed filter conditions to limit the scan range.
