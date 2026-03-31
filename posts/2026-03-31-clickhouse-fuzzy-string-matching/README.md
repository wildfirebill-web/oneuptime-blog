# How to Implement Fuzzy String Matching in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Fuzzy Search, String Matching, Analytics, Search

Description: Learn how to implement fuzzy string matching in ClickHouse using edit distance, n-gram similarity, and phonetic functions for approximate lookups.

---

Fuzzy string matching lets you find records even when the search term has typos, alternate spellings, or minor variations. ClickHouse provides several built-in functions that make this straightforward.

## Using Edit Distance

The `editDistance` function computes the Levenshtein distance between two strings - the minimum number of single-character edits needed to transform one string into another.

```sql
SELECT name
FROM users
WHERE editDistance(lower(name), lower('johnn')) <= 2;
```

This finds names within 2 edits of "johnn", catching "John", "Johnny", or "Johanna".

For case-insensitive matching, always `lower()` both sides before comparing.

## N-Gram Similarity

The `ngramSimilarity` function measures how many n-grams two strings share, returning a float between 0 and 1.

```sql
SELECT
    product_name,
    ngramSimilarity(lower(product_name), lower('sneakers')) AS score
FROM products
WHERE score > 0.3
ORDER BY score DESC
LIMIT 10;
```

N-gram similarity handles transpositions and partial matches well, making it suitable for product name lookups and user search bars.

## Combining Functions for Ranked Results

You can combine both approaches to score candidates:

```sql
SELECT
    name,
    email,
    editDistance(lower(name), lower({query:String})) AS ed,
    ngramSimilarity(lower(name), lower({query:String})) AS sim
FROM customers
WHERE ed <= 3 OR sim > 0.4
ORDER BY (ed * 0.4 + (1 - sim) * 0.6)
LIMIT 20;
```

## Phonetic Matching with soundex

For names that sound alike but are spelled differently, use `soundex`:

```sql
SELECT name
FROM customers
WHERE soundex(name) = soundex('Smyth');
-- Returns: Smith, Smythe, Smithe
```

## Practical Indexing Tips

Fuzzy matching scans are expensive. Keep performance acceptable by:

- Pre-filtering with a `LIKE '%partial%'` or bloom filter skip index
- Limiting fuzzy scan to a smaller candidate set from a prefix match
- Using materialized columns to store pre-computed soundex or ngram tokens

```sql
ALTER TABLE customers
ADD COLUMN name_soundex String MATERIALIZED soundex(name);

ALTER TABLE customers
ADD INDEX idx_name_soundex name_soundex TYPE bloom_filter GRANULARITY 1;
```

## Summary

ClickHouse supports fuzzy string matching through `editDistance`, `ngramSimilarity`, and `soundex`. Combine them to build ranked approximate search queries. To keep scans fast, pre-filter with cheaper conditions or materialize phonetic columns with skip indexes.
