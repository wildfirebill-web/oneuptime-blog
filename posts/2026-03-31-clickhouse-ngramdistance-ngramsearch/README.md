# How to Use ngramDistance() and ngramSearch() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, String Function, Fuzzy Matching, Search, SQL

Description: Learn how ngramDistance() measures string similarity using n-gram overlap and how ngramSearch() finds the closest match in an array, for fuzzy search and typo detection.

---

Exact string matching breaks down the moment data contains typos, alternate spellings, or minor variations. ClickHouse's n-gram based similarity functions provide a practical alternative. `ngramDistance()` computes a normalized distance between two strings based on shared 4-character n-grams, where `0` means the strings are identical and `1` means they share no n-grams at all. `ngramSearch()` extends this to find the most similar string in an array. Case-insensitive variants `ngramDistanceCaseInsensitive()` and `ngramSearchCaseInsensitive()` are also available.

## Function Signatures

```text
ngramDistance(str1, str2)
ngramDistanceCaseInsensitive(str1, str2)
```

Returns a `Float32` in the range `[0, 1]`. Lower values mean higher similarity.

```text
ngramSearch(str, arr)
ngramSearchCaseInsensitive(str, arr)
```

Returns the element of `arr` that is closest to `str` by n-gram distance.

## Understanding the Distance Value

A few examples illustrate how the distance scales with similarity.

```sql
SELECT
    str1,
    str2,
    ngramDistance(str1, str2) AS distance
FROM (
    SELECT * FROM VALUES(
        'str1 String, str2 String',
        ('clickhouse', 'clickhouse'),
        ('clickhouse', 'ClickHouse'),
        ('clickhouse', 'click house'),
        ('clickhouse', 'clckhouse'),
        ('clickhouse', 'postgresql'),
        ('clickhouse', 'xyz')
    )
)
```

```text
str1        | str2        | distance
------------+-------------+----------
clickhouse  | clickhouse  | 0.0
clickhouse  | ClickHouse  | 0.5
clickhouse  | click house | 0.111...
clickhouse  | clckhouse   | 0.222...
clickhouse  | postgresql  | 0.888...
clickhouse  | xyz         | 1.0
```

## Fuzzy Matching - Finding Similar Product Names

When a product catalog has slight variations in naming (extra spaces, minor misspellings), `ngramDistance` can surface near-duplicates.

```sql
SELECT
    a.product_name AS name_a,
    b.product_name AS name_b,
    ngramDistanceCaseInsensitive(a.product_name, b.product_name) AS distance
FROM products AS a
CROSS JOIN products AS b
WHERE a.product_id < b.product_id
  AND ngramDistanceCaseInsensitive(a.product_name, b.product_name) < 0.2
ORDER BY distance
LIMIT 20
```

Setting a threshold of `0.2` surfaces string pairs that are likely duplicates or near-duplicates.

## Typo Correction - Finding the Intended Search Term

Given a user's search query that may contain a typo, find the closest known term from a dictionary.

```sql
SELECT
    user_query,
    ngramSearch(user_query, ['monitoring', 'alerting', 'dashboard', 'incident', 'oncall', 'metrics']) AS closest_term,
    ngramDistance(user_query, ngramSearch(user_query, ['monitoring', 'alerting', 'dashboard', 'incident', 'oncall', 'metrics'])) AS distance
FROM (
    SELECT arrayJoin(['monitroing', 'alreting', 'dashbord', 'incidnt', 'metrcs']) AS user_query
)
```

```text
user_query   | closest_term | distance
-------------+--------------+-----------
monitroing   | monitoring   | 0.333...
alreting     | alerting     | 0.4
dashbord     | dashboard    | 0.333...
incidnt      | incident     | 0.5
metrcs       | metrics      | 0.4
```

## Filtering by Similarity Threshold

Use `ngramDistance` in a `WHERE` clause to keep only rows that are sufficiently similar to a target string.

```sql
SELECT
    company_name,
    ngramDistanceCaseInsensitive(company_name, 'Acme Corporation') AS distance
FROM companies
WHERE ngramDistanceCaseInsensitive(company_name, 'Acme Corporation') < 0.3
ORDER BY distance
LIMIT 10
```

## Ranking Search Results by Similarity

For a full-text-style search where you want to rank results by how closely they match a query term, order by `ngramDistance` ascending.

```sql
SELECT
    article_title,
    ngramDistanceCaseInsensitive(article_title, 'clickhouse performance tuning') AS distance
FROM articles
WHERE ngramDistanceCaseInsensitive(article_title, 'clickhouse performance tuning') < 0.6
ORDER BY distance
LIMIT 10
```

## Detecting Duplicate Records After Slight Formatting Changes

When records are imported from multiple sources they often carry the same address or name with minor differences. `ngramDistance` can flag these before deduplication.

```sql
SELECT
    a.id        AS id_a,
    a.address   AS address_a,
    b.id        AS id_b,
    b.address   AS address_b,
    ngramDistanceCaseInsensitive(a.address, b.address) AS distance
FROM customers AS a
CROSS JOIN customers AS b
WHERE a.id < b.id
  AND ngramDistanceCaseInsensitive(a.address, b.address) < 0.15
ORDER BY distance
LIMIT 50
```

## Combining ngramSearch with Array of Canonical Values

Normalize free-text entries by snapping them to the closest canonical value in a reference list.

```sql
SELECT
    raw_category,
    ngramSearchCaseInsensitive(
        raw_category,
        ['Infrastructure', 'Application', 'Database', 'Network', 'Security']
    ) AS canonical_category
FROM incident_reports
WHERE event_date = today()
LIMIT 50
```

## Summary

`ngramDistance()` computes a normalized `Float32` similarity score between two strings using shared 4-character n-grams, with `0` indicating identical strings and `1` indicating no overlap. `ngramSearch()` finds the closest matching string from an array. Case-insensitive variants are available for both. These functions are practical tools for fuzzy deduplication, typo correction, approximate search ranking, and snapping free-text fields to canonical values - all within a single SQL query in ClickHouse.
