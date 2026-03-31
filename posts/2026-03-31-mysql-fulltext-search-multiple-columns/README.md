# How to Search Across Multiple Columns with Full-Text Search in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Full-Text Search, InnoDB, Index, Query

Description: Learn how to create a composite full-text index and search across multiple columns simultaneously using MATCH AGAINST in MySQL.

---

## Why Search Multiple Columns?

Most content tables have meaningful text in more than one column. A blog post has a title, an excerpt, and a body. A product listing has a name, short description, and full specification. Searching only one column misses relevant matches in the others.

MySQL supports composite full-text indexes that span multiple columns, allowing a single `MATCH AGAINST` expression to search all of them simultaneously.

## Creating a Multi-Column Full-Text Index

Include all columns you want to search in a single `FULLTEXT` index:

```sql
CREATE TABLE articles (
  id INT PRIMARY KEY AUTO_INCREMENT,
  title VARCHAR(255) NOT NULL,
  excerpt VARCHAR(500),
  body TEXT,
  FULLTEXT INDEX ft_article (title, excerpt, body)
);
```

Or add a full-text index to an existing table:

```sql
ALTER TABLE articles
  ADD FULLTEXT INDEX ft_article (title, excerpt, body);
```

## Querying Multiple Columns

The `MATCH` clause must list the exact same columns as the full-text index:

```sql
SELECT id, title,
  MATCH(title, excerpt, body) AGAINST('connection pooling') AS score
FROM articles
WHERE MATCH(title, excerpt, body) AGAINST('connection pooling')
ORDER BY score DESC
LIMIT 10;
```

If the column list in `MATCH` does not exactly match an existing full-text index, MySQL returns an error.

## Column Order Matters

MySQL weights matches in earlier columns slightly higher. Placing your most important column first (typically the title) gives title matches a relevance boost.

```sql
-- title is first - matches in the title score higher
FULLTEXT INDEX ft_article (title, excerpt, body)
```

## Boolean Mode with Multiple Columns

Boolean mode operators work the same way with multi-column indexes:

```sql
-- Require 'mysql' and optionally boost 'replication'
SELECT id, title
FROM articles
WHERE MATCH(title, excerpt, body)
  AGAINST('+mysql replication -deprecated' IN BOOLEAN MODE)
ORDER BY MATCH(title, excerpt, body)
  AGAINST('+mysql replication -deprecated' IN BOOLEAN MODE) DESC;
```

## You Cannot Mix Indexes in One MATCH

A single `MATCH` expression can only use one full-text index. You cannot combine columns from different indexes:

```sql
-- ERROR: cannot mix columns from different full-text indexes
SELECT * FROM articles
WHERE MATCH(title) AGAINST('query')
   OR MATCH(body) AGAINST('query');
```

The correct approach is to define one composite index and use it:

```sql
-- CORRECT: one composite index covers all columns
SELECT * FROM articles
WHERE MATCH(title, body) AGAINST('query');
```

## Separate Indexes vs Composite Index

If you need to search subsets of columns independently, create separate indexes:

```sql
ALTER TABLE articles ADD FULLTEXT INDEX ft_title (title);
ALTER TABLE articles ADD FULLTEXT INDEX ft_body (body);
ALTER TABLE articles ADD FULLTEXT INDEX ft_all (title, excerpt, body);
```

Each index serves a different query pattern. Use the composite index for broad searches, individual indexes for targeted column searches.

## Checking Which Index Is Used

```sql
EXPLAIN SELECT * FROM articles
WHERE MATCH(title, excerpt, body) AGAINST('performance tuning');
```

The output should show `key: ft_article` confirming the composite full-text index is selected.

## Summary

Multi-column full-text search in MySQL requires a composite `FULLTEXT` index covering all target columns. The `MATCH` clause must list the exact same columns as the index. Earlier columns receive slightly higher relevance weighting. For flexible querying, you can create both composite and single-column full-text indexes on the same table to serve different search patterns.
