# How to Use FIND_IN_SET() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, String Function, Database

Description: Learn how FIND_IN_SET() searches for a value in a comma-separated string in MySQL, its use with SET columns, limitations compared to normalized tables, and performance considerations.

---

`FIND_IN_SET(str, strlist)` returns the position (1-indexed) of `str` within a comma-separated list `strlist`. It is primarily used with MySQL's `SET` data type and comma-delimited string columns.

## Syntax

```sql
FIND_IN_SET(str, strlist)
```

- `str` - the value to search for.
- `strlist` - a comma-separated string such as `'a,b,c,d'`.
- Returns the position (1 to N) if found, 0 if not found.
- Returns `NULL` if either argument is `NULL`.
- `str` must not contain a comma. `strlist` uses commas as delimiters with no spaces.

## Basic usage

```sql
SELECT FIND_IN_SET('b', 'a,b,c,d');   -- 2
SELECT FIND_IN_SET('d', 'a,b,c,d');   -- 4
SELECT FIND_IN_SET('e', 'a,b,c,d');   -- 0  (not found)
SELECT FIND_IN_SET('a', '');           -- 0
SELECT FIND_IN_SET(NULL, 'a,b,c');    -- NULL
```

## Using with a SET column

MySQL's `SET` column stores a selection of predefined values. `FIND_IN_SET` is the natural way to search within a `SET` column:

```sql
CREATE TABLE user_preferences (
    user_id      INT PRIMARY KEY,
    name         VARCHAR(100),
    notifications SET('email', 'sms', 'push', 'in_app')
);

INSERT INTO user_preferences VALUES
(1, 'Alice', 'email,push'),
(2, 'Bob',   'sms,email,in_app'),
(3, 'Carol', 'push');

-- Find users who have email notifications enabled
SELECT name FROM user_preferences WHERE FIND_IN_SET('email', notifications) > 0;
-- Alice, Bob

-- Find users who have sms notifications enabled
SELECT name FROM user_preferences WHERE FIND_IN_SET('sms', notifications) > 0;
-- Bob
```

## Using with a delimited string column

```sql
CREATE TABLE articles (
    article_id INT PRIMARY KEY,
    title      VARCHAR(200),
    tags       VARCHAR(200)  -- e.g. 'mysql,database,sql'
);

INSERT INTO articles VALUES
(1, 'MySQL Basics',    'mysql,database,sql'),
(2, 'PostgreSQL Tips', 'postgresql,database'),
(3, 'Redis Caching',   'redis,cache,nosql');

-- Find articles tagged with 'database'
SELECT title FROM articles WHERE FIND_IN_SET('database', tags) > 0;
-- MySQL Basics, PostgreSQL Tips
```

## Getting the position

```sql
SELECT FIND_IN_SET('mysql', 'redis,mysql,postgresql') AS position;
-- 2
```

Use the returned position with other string functions or for sorting:

```sql
-- Sort results by the position of a specific tag (members of the 'mysql' group first)
SELECT title, FIND_IN_SET('mysql', tags) AS tag_pos
FROM articles
ORDER BY FIND_IN_SET('mysql', tags) = 0, FIND_IN_SET('mysql', tags);
```

## Checking membership in a fixed list

```sql
-- Check if a status value belongs to an active set
SELECT order_id, status
FROM orders
WHERE FIND_IN_SET(status, 'shipped,delivered,completed') > 0;

-- Often simpler with IN:
WHERE status IN ('shipped', 'delivered', 'completed');
```

Use `IN` for static lists. Use `FIND_IN_SET` when the list itself is a column value or a parameter.

## Comparing FIND_IN_SET with LIKE

```sql
-- LIKE approach (matches partial substrings, fragile)
WHERE tags LIKE '%mysql%';  -- also matches 'mysql8', 'premysql'

-- FIND_IN_SET approach (exact token match)
WHERE FIND_IN_SET('mysql', tags) > 0;
```

`FIND_IN_SET` avoids false matches from partial substrings.

## Limitations

| Limitation | Detail |
|---|---|
| No index usage | `FIND_IN_SET` on a column cannot use a standard index |
| No spaces around commas | `'a, b, c'` treats `' b'` and `' c'` as the values, not `'b'` and `'c'` |
| Comma in str not allowed | `FIND_IN_SET('a,b', 'a,b,c')` searches for the literal `'a,b'` token |
| Performance | Full table scan for large tables |

For frequently searched multi-valued attributes, use a normalised junction table instead:

```sql
-- Normalised alternative
CREATE TABLE article_tags (
    article_id INT,
    tag        VARCHAR(50),
    PRIMARY KEY (article_id, tag)
);

CREATE INDEX idx_article_tags_tag ON article_tags (tag);

-- Fast indexed query
SELECT a.title FROM articles a
INNER JOIN article_tags t ON a.article_id = t.article_id
WHERE t.tag = 'mysql';
```

## Summary

`FIND_IN_SET(str, strlist)` returns the position of `str` in a comma-separated list, or 0 if not found. It is the canonical function for querying MySQL `SET` columns and comma-delimited string fields. It provides exact token matching, unlike `LIKE`. However, it cannot use indexes and is slow on large tables. For any multi-valued attribute that is frequently searched, normalise it into a junction table with a proper index.
