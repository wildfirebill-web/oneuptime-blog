# How to Use length() and empty() String Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, String Function, SQL, UTF-8

Description: length() returns the byte length of a string, lengthUTF8() returns the character count, and empty() checks for zero-length strings. Includes multi-byte UTF-8 examples.

---

When working with string columns in ClickHouse, you often need to know how long a value is or whether it contains anything at all. ClickHouse provides three related functions for this: `length()`, `lengthUTF8()`, and `empty()`. Understanding when to use each one is important because `length()` counts bytes, not characters, which means it gives unexpected results for multi-byte UTF-8 text.

## length() - Byte Length

`length(str)` returns the number of bytes in the string. For ASCII strings (where every character is one byte), this equals the number of characters. For strings containing multi-byte UTF-8 characters such as accented letters, Cyrillic, Chinese, or emoji, the byte count will be larger than the character count.

```sql
SELECT
    'hello'          AS str,
    length('hello')  AS byte_length;
```

```text
str   | byte_length
------+------------
hello | 5
```

A purely ASCII string of 5 characters has a byte length of 5.

## length() with Multi-Byte Characters

The difference between bytes and characters becomes clear with non-ASCII content.

```sql
SELECT
    str,
    length(str)      AS byte_length,
    lengthUTF8(str)  AS char_length
FROM (
    SELECT 'hello'      AS str UNION ALL
    SELECT 'café'       AS str UNION ALL
    SELECT 'привет'     AS str UNION ALL
    SELECT '你好'        AS str UNION ALL
    SELECT '🚀'         AS str
);
```

```text
str    | byte_length | char_length
-------+-------------+------------
hello  | 5           | 5
café   | 5           | 4
привет | 12          | 6
你好    | 6           | 2
🚀     | 4           | 1
```

"café" has 4 characters but 5 bytes because the accented é is two bytes in UTF-8. "привет" (Russian for "hello") has 6 characters but 12 bytes. The rocket emoji is 1 character but 4 bytes. Use `lengthUTF8()` whenever you care about visible character count rather than storage size.

## lengthUTF8() for Character Count

`lengthUTF8(str)` counts Unicode code points (characters). Use this when you need to enforce a character limit on user-facing text such as usernames, tweet-like messages, or display labels.

```sql
SELECT
    username,
    lengthUTF8(username) AS char_count,
    CASE
        WHEN lengthUTF8(username) > 20 THEN 'Too long'
        WHEN lengthUTF8(username) < 3  THEN 'Too short'
        ELSE 'Valid'
    END AS validation_result
FROM (
    SELECT 'alice'                     AS username UNION ALL
    SELECT 'ab'                        AS username UNION ALL
    SELECT 'this_username_is_way_too_long_for_display' AS username UNION ALL
    SELECT 'пользователь'              AS username
);
```

## empty() - Checking for Zero-Length Strings

`empty(str)` returns 1 if the string has zero length, and 0 otherwise. It is equivalent to `length(str) = 0` but more readable and slightly more efficient because it does not need to compute the full length.

```sql
SELECT
    str,
    empty(str)    AS is_empty,
    notEmpty(str) AS is_not_empty
FROM (
    SELECT ''       AS str UNION ALL
    SELECT ' '      AS str UNION ALL
    SELECT 'hello'  AS str UNION ALL
    SELECT '\0'     AS str
);
```

```text
str   | is_empty | is_not_empty
------+----------+-------------
      | 1        | 0
      | 0        | 1
hello | 0        | 1
\0    | 0        | 1
```

Note that a string containing only a space character is not empty - it has one byte. `empty()` only considers strings with zero bytes as empty.

## Filtering Rows with empty() and notEmpty()

Use `empty()` and `notEmpty()` in WHERE clauses to exclude blank values from analysis.

```sql
CREATE TABLE user_profiles
(
    user_id  UInt32,
    username String,
    bio      String
)
ENGINE = MergeTree()
ORDER BY user_id;

INSERT INTO user_profiles VALUES
    (1, 'alice',  'Software engineer and coffee enthusiast'),
    (2, 'bob',    ''),
    (3, 'claire', 'Product designer'),
    (4, 'dan',    ''),
    (5, 'eve',    'Security researcher');
```

```sql
-- Count users with and without a bio
SELECT
    countIf(notEmpty(bio)) AS has_bio,
    countIf(empty(bio))    AS no_bio
FROM user_profiles;
```

```sql
-- Return only users who have filled in their bio
SELECT user_id, username, bio
FROM user_profiles
WHERE notEmpty(bio)
ORDER BY user_id;
```

## Using length() for Data Quality Checks

Check that critical string columns fall within expected byte ranges.

```sql
SELECT
    user_id,
    username,
    length(username) AS username_bytes,
    CASE
        WHEN length(username) = 0  THEN 'Empty'
        WHEN length(username) < 3  THEN 'Too short'
        WHEN length(username) > 50 THEN 'Too long'
        ELSE 'OK'
    END AS username_status
FROM user_profiles
ORDER BY user_id;
```

## Combining length() and lengthUTF8() to Detect Multi-Byte Content

When `length(str) > lengthUTF8(str)`, the string contains at least one multi-byte character. This is useful for routing strings to UTF-8 aware processing paths.

```sql
SELECT
    str,
    length(str)     AS bytes,
    lengthUTF8(str) AS chars,
    bytes > chars   AS has_multibyte
FROM (
    SELECT 'hello'     AS str UNION ALL
    SELECT 'hëllo'     AS str UNION ALL
    SELECT 'データ'     AS str
);
```

## Summary

`length()` counts bytes and is fastest for ASCII-only data. `lengthUTF8()` counts Unicode characters and is correct for any internationalized content. `empty()` efficiently tests whether a string has zero length, with its complement `notEmpty()` useful in WHERE and countIf expressions. For any text that may contain characters from non-Latin scripts, prefer `lengthUTF8()` over `length()` when the purpose is validating visible character count rather than measuring storage footprint.
