# How to Use tokens() Function in ClickHouse for Text Tokenization

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, String Function, Full-Text Search, Array, SQL

Description: Learn how tokens() splits strings into arrays of alphanumeric words in ClickHouse, the same tokenization used by full-text search indexes for vocabulary analysis.

---

Full-text search in ClickHouse is built on top of token-based indexes. When you create a `tokenbf_v1` or `full_text` index on a column, ClickHouse internally splits each value into tokens using the same logic exposed by the `tokens()` function. Understanding `tokens()` directly lets you explore what the index sees, build vocabulary statistics, filter by token presence, and pre-process text data for downstream analysis.

## How tokens() Works

`tokens(str)` splits a string on any non-alphanumeric character and returns the resulting alphanumeric segments as an array of strings. Separators (spaces, punctuation, slashes, dots, etc.) are discarded. Empty segments are not included.

```sql
-- Basic tokenization
SELECT tokens('Hello, world! How are you?') AS word_tokens;
```

```text
word_tokens
----------------------------------
['Hello', 'world', 'How', 'are', 'you']
```

```sql
-- Tokenizing a URL path
SELECT tokens('https://example.com/api/v2/users?page=1&limit=50') AS url_tokens;
```

```text
url_tokens
--------------------------------------------------
['https', 'example', 'com', 'api', 'v2', 'users', 'page', '1', 'limit', '50']
```

Every non-alphanumeric character acts as a delimiter. Numbers are kept as tokens.

## Tokenizing Search Queries

A common use case is splitting a user-supplied search query into individual terms to apply multi-column OR logic.

```sql
-- Tokenize a search phrase and check each article for any matching token
SELECT
    article_id,
    title,
    hasAny(tokens(lower(title)), tokens(lower('machine learning algorithms'))) AS title_matches
FROM articles
WHERE title_matches = 1
LIMIT 10;
```

`hasAny(array1, array2)` returns `1` if the two arrays share at least one element. Applying `lower()` before `tokens()` makes the comparison case-insensitive.

## Counting Vocabulary Size

Use `tokens()` with array aggregation functions to compute vocabulary statistics across a corpus.

```sql
-- Count distinct tokens across all document titles
SELECT
    length(groupUniqArray(token)) AS vocabulary_size
FROM (
    SELECT arrayJoin(tokens(lower(title))) AS token
    FROM articles
);
```

```sql
-- Find the 20 most frequent tokens in article bodies
SELECT
    token,
    count() AS frequency
FROM (
    SELECT arrayJoin(tokens(lower(body))) AS token
    FROM articles
    WHERE length(body) > 0
)
WHERE length(token) > 3  -- skip very short tokens
GROUP BY token
ORDER BY frequency DESC
LIMIT 20;
```

## Filtering Rows by Token Presence

Because `tokens()` returns an array, you can use `has()` to check for a specific token. This is semantically different from `LIKE '%word%'` because `tokens()` only matches on word boundaries.

```sql
-- Find articles that contain the exact token "clickhouse"
SELECT
    article_id,
    title
FROM articles
WHERE has(tokens(lower(body)), 'clickhouse')
LIMIT 20;
```

```sql
-- Contrast with LIKE which would also match "clickhouse2023" as containing "clickhouse"
-- tokens() only returns 'clickhouse2023' as a single token, not split further
SELECT tokens('clickhouse2023 vs clickhouse 24') AS result;
```

```text
result
----------------------------------------------
['clickhouse2023', 'vs', 'clickhouse', '24']
```

## Comparing Token Overlap Between Strings

Token overlap is a simple approximation of text similarity. You can compute it by comparing the union and intersection of two token arrays.

```sql
-- Compute Jaccard similarity between two text fields
SELECT
    length(arrayIntersect(tokens(lower(title)), tokens(lower(description)))) AS intersection_size,
    length(arrayDistinct(arrayConcat(tokens(lower(title)), tokens(lower(description))))) AS union_size,
    intersection_size / union_size AS jaccard_similarity
FROM articles
WHERE union_size > 0
LIMIT 10;
```

## Building an Inverted Index Manually

If you want to understand how a token-based index works, you can materialize the token array and use it for lookups.

```sql
-- Store tokens as a materialized column for fast array lookups
ALTER TABLE articles
ADD COLUMN body_tokens Array(String)
MATERIALIZED tokens(lower(body));
```

```sql
-- After populating the materialized column, filter on it
SELECT article_id, title
FROM articles
WHERE has(body_tokens, 'opentelemetry')
LIMIT 10;
```

## Generating N-grams from Tokens

Combine `tokens()` with array manipulation to generate bigrams or trigrams for phrase-level analysis.

```sql
-- Generate bigrams from a tokenized sentence
SELECT
    arrayMap(
        i -> concat(tokens_arr[i], ' ', tokens_arr[i + 1]),
        range(1, length(tokens_arr))
    ) AS bigrams
FROM (
    SELECT tokens('the quick brown fox jumps over the lazy dog') AS tokens_arr
);
```

```text
bigrams
------------------------------------------------------------------------
['the quick', 'quick brown', 'brown fox', 'fox jumps', 'jumps over', 'over the', 'the lazy', 'lazy dog']
```

## Summary

`tokens()` exposes the same tokenization logic that powers ClickHouse full-text search indexes, making it a powerful tool for vocabulary analysis, document filtering, and text similarity computation. It splits strings on non-alphanumeric boundaries and returns a clean array of word tokens. Use it with `has()` for exact token lookups, `hasAny()` for multi-keyword matching, `arrayJoin()` for per-token aggregations, and `arrayIntersect()` for overlap-based similarity scoring.
