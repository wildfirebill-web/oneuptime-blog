# How to Use splitByRegexp() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, String Function, Regex, Array, Tokenization

Description: Learn how splitByRegexp() splits strings on regex-defined delimiters in ClickHouse, with examples for multi-delimiter splitting and whitespace normalization.

---

`splitByChar()` and `splitByString()` cover the common case where the delimiter is a fixed, known string. When the delimiter itself varies - for example, any run of whitespace, any punctuation character, or one of several different separators - you need `splitByRegexp()`. It splits a string wherever the regular expression matches, returning the non-matching parts as an `Array(String)`.

## Function Signature

```text
splitByRegexp(pattern, str)
```

`pattern` is a RE2-compatible regular expression. Everywhere the pattern matches in `str`, a split occurs. The matched delimiter text is discarded; only the segments between matches are returned.

```sql
-- Split on any whitespace sequence
SELECT splitByRegexp('\\s+', 'hello   world\ttab\nnewline') AS tokens;
-- result: ['hello', 'world', 'tab', 'newline']
```

```sql
-- Split on a comma optionally surrounded by spaces
SELECT splitByRegexp('\\s*,\\s*', 'one , two ,three,  four') AS parts;
-- result: ['one', 'two', 'three', 'four']
```

## Splitting on Multiple Delimiters

A common scenario is input that uses different separator characters interchangeably. `splitByRegexp()` handles this with a single alternation pattern:

```sql
-- Split on comma, semicolon, or pipe
SELECT splitByRegexp('[,;|]', 'alpha,beta;gamma|delta') AS parts;
-- result: ['alpha', 'beta', 'gamma', 'delta']
```

```sql
-- Split a compound identifier on any non-alphanumeric character
SELECT splitByRegexp('[^a-zA-Z0-9]+', 'com.example/api-v2_service') AS parts;
-- result: ['com', 'example', 'api', 'v2', 'service']
```

## Whitespace Normalization and Tokenization

When processing natural language text or log messages, tokenizing on whitespace is often the first step. Using `\\s+` instead of a single space correctly handles tabs, newlines, and consecutive spaces:

```sql
-- Tokenize a log message into individual words
SELECT
    message_id,
    splitByRegexp('\\s+', trim(lower(message))) AS tokens
FROM log_messages
LIMIT 5;
```

```sql
-- Count word frequency across all support tickets
SELECT
    word,
    COUNT(*) AS occurrences
FROM (
    SELECT arrayJoin(
        splitByRegexp('\\s+', trim(lower(body)))
    ) AS word
    FROM support_tickets
    WHERE body != ''
)
WHERE length(word) > 3
GROUP BY word
ORDER BY occurrences DESC
LIMIT 30;
```

## Advanced Tokenization for Search Indexing

Splitting on non-word characters produces clean tokens suitable for building inverted index-style aggregations:

```sql
-- Extract all alphanumeric tokens from a description field
SELECT
    product_id,
    arrayFilter(
        t -> length(t) > 0,
        splitByRegexp('[^a-zA-Z0-9]', description)
    ) AS tokens
FROM products
LIMIT 10;
```

```sql
-- Find documents that share at least 5 tokens with a given query string
WITH splitByRegexp('[^a-zA-Z0-9]+', lower('clickhouse string functions tutorial')) AS query_tokens
SELECT
    doc_id,
    title,
    length(arrayIntersect(
        splitByRegexp('[^a-zA-Z0-9]+', lower(content)),
        query_tokens
    )) AS shared_tokens
FROM documents
HAVING shared_tokens >= 5
ORDER BY shared_tokens DESC
LIMIT 10;
```

## Splitting Version Strings

Version strings frequently use dots, hyphens, or plus signs as separators. A single `splitByRegexp()` call handles any combination:

```sql
-- Parse semantic version strings with various separator styles
SELECT
    version_string,
    splitByRegexp('[.\\-+]', version_string) AS parts,
    splitByRegexp('[.\\-+]', version_string)[1] AS major,
    splitByRegexp('[.\\-+]', version_string)[2] AS minor,
    splitByRegexp('[.\\-+]', version_string)[3] AS patch
FROM releases
LIMIT 10;
```

## Parsing Key-Value Pairs

Some log formats embed key=value pairs separated by varying whitespace or punctuation. `splitByRegexp()` can be the first step in parsing them:

```sql
-- Split a log attribute string into individual key=value tokens
SELECT
    log_id,
    splitByRegexp('\\s+', trim(attributes)) AS kv_tokens
FROM structured_logs
LIMIT 5;
```

```sql
-- Extract values for a specific key from each kv token list
SELECT
    log_id,
    arrayFirst(
        x -> x LIKE 'status=%',
        splitByRegexp('\\s+', trim(attributes))
    ) AS status_token
FROM structured_logs
LIMIT 10;
```

## Handling Edge Cases

When the pattern matches at the very start or end of the string, an empty string element is produced at the corresponding end of the array:

```sql
SELECT splitByRegexp(',', ',a,b,') AS result;
-- result: ['', 'a', 'b', '']
```

Filter out empty elements with `arrayFilter()` when you do not want them:

```sql
SELECT arrayFilter(x -> x != '', splitByRegexp(',', ',a,b,')) AS result;
-- result: ['a', 'b']
```

When the pattern does not match anywhere in the string, the entire string is returned as a single-element array:

```sql
SELECT splitByRegexp('\\d+', 'no digits here') AS result;
-- result: ['no digits here']
```

## Combining with Array Functions

The returned `Array(String)` integrates with the full set of ClickHouse array functions:

```sql
-- Count the number of tokens in each message
SELECT
    message_id,
    length(splitByRegexp('\\s+', trim(message))) AS word_count
FROM messages
ORDER BY word_count DESC
LIMIT 10;
```

```sql
-- Check if a tokenized path contains a specific segment
SELECT
    url_path,
    has(splitByRegexp('[/?&=]', url_path), 'admin') AS is_admin_path
FROM page_views
LIMIT 10;
```

## Summary

`splitByRegexp(pattern, str)` splits a string wherever the RE2 pattern matches, discarding the matched delimiter text and returning the segments as `Array(String)`. It handles variable delimiters, multi-character separators, and whitespace normalization scenarios that `splitByChar()` and `splitByString()` cannot express cleanly. Use `arrayFilter()` to remove empty strings produced by leading or trailing delimiters. The resulting array works directly with all standard ClickHouse array functions.
