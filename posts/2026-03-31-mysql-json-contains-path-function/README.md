# How to Use JSON_CONTAINS_PATH() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, JSON, Database, Function

Description: Learn how to use MySQL JSON_CONTAINS_PATH() to test whether a JSON document contains one or more specified paths, using 'one' or 'all' mode flags.

---

## What JSON_CONTAINS_PATH() Does

`JSON_CONTAINS_PATH()` checks whether a JSON document has data at one or more given paths. It returns 1 if the condition is satisfied and 0 otherwise. It does not check the value at the path - only whether the path itself exists.

```sql
SELECT JSON_CONTAINS_PATH('{"a": 1, "b": 2}', 'one', '$.a');
-- 1  (path $.a exists)

SELECT JSON_CONTAINS_PATH('{"a": 1}', 'one', '$.c');
-- 0  (path $.c does not exist)
```

The function signature is:

```sql
JSON_CONTAINS_PATH(json_doc, one_or_all, path [, path ...])
```

- `json_doc` - the JSON value to inspect
- `one_or_all` - either `'one'` or `'all'`
- `path` - one or more JSONPath strings

## The one vs all Mode

| Mode | Returns 1 when |
|---|---|
| `'one'` | At least one of the paths exists |
| `'all'` | Every path exists |

```sql
SET @doc = '{"name": "Alice", "age": 30}';

-- 'one': returns 1 because $.name exists (even though $.email does not)
SELECT JSON_CONTAINS_PATH(@doc, 'one', '$.name', '$.email');
-- 1

-- 'all': returns 0 because $.email is missing
SELECT JSON_CONTAINS_PATH(@doc, 'all', '$.name', '$.email');
-- 0

-- 'all': returns 1 because both $.name and $.age exist
SELECT JSON_CONTAINS_PATH(@doc, 'all', '$.name', '$.age');
-- 1
```

## Using with a Table

```sql
CREATE TABLE user_profiles (
  id      INT AUTO_INCREMENT PRIMARY KEY,
  profile JSON NOT NULL
);

INSERT INTO user_profiles (profile) VALUES
  ('{"name": "Alice", "age": 28, "address": {"city": "Oslo"}}'),
  ('{"name": "Bob",   "age": 34}'),
  ('{"name": "Carol", "age": 22, "phone": "555-0100"}');
```

Find profiles that have an address:

```sql
SELECT id, profile->>'$.name' AS name
FROM user_profiles
WHERE JSON_CONTAINS_PATH(profile, 'one', '$.address') = 1;
```

```text
1  Alice
```

Find profiles that have both age and phone:

```sql
SELECT id, profile->>'$.name' AS name
FROM user_profiles
WHERE JSON_CONTAINS_PATH(profile, 'all', '$.age', '$.phone') = 1;
```

```text
3  Carol
```

## Checking Nested Paths

```sql
-- Does the profile have a city inside address?
SELECT id, profile->>'$.name' AS name
FROM user_profiles
WHERE JSON_CONTAINS_PATH(profile, 'one', '$.address.city') = 1;
```

## Checking Array Elements

Use `[*]` to check whether an array contains any element, or `[0]` to check a specific index:

```sql
SET @doc = '{"tags": ["sale", "new"]}';

-- Does $.tags exist and has any element?
SELECT JSON_CONTAINS_PATH(@doc, 'one', '$.tags[0]');  -- 1
SELECT JSON_CONTAINS_PATH(@doc, 'one', '$.tags[5]');  -- 0 (index 5 does not exist)
```

## Difference from JSON_CONTAINS()

`JSON_CONTAINS_PATH()` checks whether a path exists. `JSON_CONTAINS()` checks whether a specific value is present anywhere in the document or at a given path:

```sql
-- JSON_CONTAINS_PATH: does $.age exist?
SELECT JSON_CONTAINS_PATH('{"age": 30}', 'one', '$.age');  -- 1

-- JSON_CONTAINS: is value 30 at $.age?
SELECT JSON_CONTAINS('{"age": 30}', '30', '$.age');  -- 1

-- JSON_CONTAINS: is value 99 at $.age?
SELECT JSON_CONTAINS('{"age": 30}', '99', '$.age');  -- 0
```

Use `JSON_CONTAINS_PATH()` when you only need to confirm a key exists, regardless of its value. Use `JSON_CONTAINS()` when you need to check for a specific value.

## Finding Records Missing a Required Field

Invert the check to find rows where a required JSON key is absent:

```sql
SELECT id, profile->>'$.name' AS name
FROM user_profiles
WHERE JSON_CONTAINS_PATH(profile, 'one', '$.address') = 0;
```

This is useful for data quality checks and backfill scripts.

## Summary

`JSON_CONTAINS_PATH(json_doc, mode, path ...)` returns 1 if the specified paths exist in the document. Use `'one'` mode to check for any of the listed paths, and `'all'` mode to require every path to be present. Combine with `= 0` to find records missing a field. For value-level checks, use `JSON_CONTAINS()` instead.
