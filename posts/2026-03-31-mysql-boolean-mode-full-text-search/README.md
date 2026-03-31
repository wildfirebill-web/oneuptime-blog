# How to Use Boolean Mode Full-Text Search in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Full-Text Search, Boolean Mode, FULLTEXT, InnoDB

Description: Learn how to use Boolean mode full-text search in MySQL with MATCH AGAINST IN BOOLEAN MODE for precise, operator-driven text search queries.

---

## What Is Boolean Mode Full-Text Search?

MySQL full-text search supports two main modes: natural language mode and Boolean mode. Boolean mode gives you direct control over how search terms are matched using operators like `+` (required), `-` (excluded), `*` (wildcard), `"..."` (phrase), and `~` (lower relevance).

In Boolean mode, results are not ranked by default, but the query gives you precise control over inclusion and exclusion of terms.

## Creating a FULLTEXT Index

Boolean mode requires a FULLTEXT index on the columns being searched:

```sql
CREATE TABLE articles (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(255),
    body TEXT,
    FULLTEXT idx_ft (title, body)
);
```

Or add it to an existing table:

```sql
ALTER TABLE articles ADD FULLTEXT INDEX idx_ft (title, body);
```

## Basic Boolean Mode Query

```sql
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('mysql performance' IN BOOLEAN MODE);
```

Without operators, this works similarly to natural language mode: rows containing either word are returned.

## The + Operator: Require a Term

The `+` prefix makes a term mandatory - all results must contain it:

```sql
-- Results must contain both 'mysql' AND 'index'
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('+mysql +index' IN BOOLEAN MODE);
```

## The - Operator: Exclude a Term

The `-` prefix excludes rows that contain the term:

```sql
-- Must contain 'mysql', must NOT contain 'deprecated'
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('+mysql -deprecated' IN BOOLEAN MODE);
```

## Phrase Search with Quotes

Use double quotes to search for exact phrases:

```sql
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('"query optimization"' IN BOOLEAN MODE);
```

## The * Wildcard: Prefix Matching

The `*` suffix matches any word starting with the prefix:

```sql
-- Matches 'partition', 'partitioning', 'partitioned'
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('partition*' IN BOOLEAN MODE);
```

## The ~ Operator: Reduce Relevance

The `~` prefix includes the term but lowers its relevance contribution:

```sql
-- Prefer 'performance' results, but include 'optimization' with lower rank
SELECT id, title, MATCH(title, body) AGAINST ('performance ~optimization' IN BOOLEAN MODE) AS score
FROM articles
WHERE MATCH(title, body) AGAINST ('performance ~optimization' IN BOOLEAN MODE)
ORDER BY score DESC;
```

## Combining Operators

```sql
-- Must have 'mysql', must have 'replication' or 'cluster',
-- must NOT mention 'deprecated'
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST (
    '+mysql +(replication cluster) -deprecated'
    IN BOOLEAN MODE
);
```

## Boolean Mode Does Not Require FULLTEXT Index Always

For MyISAM tables with no FULLTEXT index, boolean mode searches still work but will be slow. Always create the FULLTEXT index for production use.

## Checking the Index

```sql
SHOW INDEX FROM articles WHERE Index_type = 'FULLTEXT';
```

## Summary

Boolean mode full-text search in MySQL gives you operator-driven control over search logic that natural language mode cannot provide. Use `+` for required terms, `-` for exclusions, `"..."` for phrase matching, and `*` for prefix wildcards. It is the right mode when users need precise, structured search queries rather than ranked relevance scoring.
