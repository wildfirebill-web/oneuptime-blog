# How to Use JSON_OBJECT() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, JSON, Function

Description: Learn how to use MySQL's JSON_OBJECT() function to construct JSON objects from key-value pairs, with examples for dynamic document creation.

---

## What Is JSON_OBJECT()?

`JSON_OBJECT()` creates a JSON object from a list of key-value pairs. Keys must be strings (or values castable to strings), and values can be any SQL expression. The function returns a `JSON`-typed value and was introduced in MySQL 5.7.8.

Syntax:

```sql
JSON_OBJECT(key1, val1, key2, val2, ...)
```

Calling it with no arguments returns an empty JSON object `{}`.

## Basic Usage

```sql
SELECT JSON_OBJECT('name', 'Alice', 'age', 30, 'active', TRUE);
-- Result: {"age": 30, "name": "Alice", "active": true}

SELECT JSON_OBJECT('score', 98.5, 'rank', NULL);
-- Result: {"rank": null, "score": 98.5}
```

Keys in the output are sorted alphabetically - MySQL normalizes JSON object key ordering.

## Using Column Values as Keys or Values

You can use column references for both keys and values:

```sql
SELECT JSON_OBJECT(
  'id', user_id,
  'name', CONCAT(first_name, ' ', last_name),
  'email', email,
  'registered', DATE_FORMAT(created_at, '%Y-%m-%d')
) AS user_json
FROM users
WHERE active = 1;
```

This pattern is common when building REST API responses directly from SQL.

## Handling Duplicate Keys

If you provide duplicate keys, MySQL keeps the last value for that key:

```sql
SELECT JSON_OBJECT('a', 1, 'a', 2);
-- Result: {"a": 2}
```

Avoid relying on this behavior - treat it as undefined and ensure keys are unique.

## Nesting JSON_OBJECT() with JSON_ARRAY()

Complex documents can be built by nesting functions:

```sql
SELECT JSON_OBJECT(
  'user_id', id,
  'profile', JSON_OBJECT(
    'name', name,
    'bio', bio
  ),
  'roles', JSON_ARRAY('admin', 'editor')
) AS document
FROM users
WHERE id = 42;
```

## Storing Objects in a JSON Column

```sql
CREATE TABLE events (
  id INT AUTO_INCREMENT PRIMARY KEY,
  event_name VARCHAR(100),
  metadata JSON
);

INSERT INTO events (event_name, metadata)
VALUES (
  'user_login',
  JSON_OBJECT('ip', '192.168.1.1', 'user_agent', 'Mozilla/5.0', 'timestamp', UNIX_TIMESTAMP())
);
```

Query the stored object using path expressions:

```sql
SELECT event_name, metadata->>'$.ip' AS ip_address
FROM events
WHERE event_name = 'user_login';
```

## NULL Key Behavior

If a key is `NULL`, MySQL raises an error:

```sql
SELECT JSON_OBJECT(NULL, 'value');
-- ERROR 3158 (22032): JSON documents may not contain NULL member names.
```

Always ensure keys are non-null when building objects dynamically.

## Combining with JSON_MERGE_PATCH() for Updates

```sql
UPDATE users
SET preferences = JSON_MERGE_PATCH(
  preferences,
  JSON_OBJECT('theme', 'dark', 'language', 'en')
)
WHERE id = 42;
```

This merges the new object into the existing preferences document, overwriting only matching keys.

## Summary

`JSON_OBJECT()` is the primary way to construct JSON documents inline in MySQL queries. It maps key-value pairs to a typed JSON object, supports nesting with other JSON functions, and integrates naturally with JSON column storage. Use it whenever you need to serialize rows into JSON format or build structured payloads from SQL expressions.
