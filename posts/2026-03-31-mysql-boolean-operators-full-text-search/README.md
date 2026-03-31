# How to Use Boolean Operators in MySQL Full-Text Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Full-Text Search, Boolean Mode, FULLTEXT, Search Operator

Description: Learn how to use Boolean operators (+, -, *, ~, >, <) in MySQL full-text search Boolean mode for precise and flexible text matching queries.

---

## Overview of Boolean Operators

MySQL full-text search in Boolean mode (`IN BOOLEAN MODE`) supports a set of prefix and modifier operators that give you fine-grained control over which documents match and how they are ranked. These operators are placed directly in the search string.

## The Full Operator Reference

| Operator | Name | Effect |
|---|---|---|
| `+` | Required | Row must contain the term |
| `-` | Prohibited | Row must NOT contain the term |
| `>` | Boost | Include and increase relevance |
| `<` | Lower | Include and decrease relevance |
| `~` | Negate | Include but reduce relevance (makes term a noise term) |
| `*` | Wildcard | Match any word with this prefix |
| `"..."` | Phrase | Match the exact phrase |
| `()` | Grouping | Group terms into sub-expressions |

## Setup

```sql
CREATE TABLE articles (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(255),
    body TEXT,
    FULLTEXT idx_ft (title, body)
);
```

## + Required Operator

```sql
-- Both terms must be present
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('+mysql +replication' IN BOOLEAN MODE);
```

## - Prohibited Operator

```sql
-- Must contain 'mysql', must NOT contain 'deprecated'
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('+mysql -deprecated' IN BOOLEAN MODE);
```

## > and < Relevance Boost Operators

```sql
-- 'performance' is more important than 'tuning' in ranking
SELECT id, title,
    MATCH(title, body) AGAINST ('>performance <tuning' IN BOOLEAN MODE) AS score
FROM articles
WHERE MATCH(title, body) AGAINST ('>performance <tuning' IN BOOLEAN MODE)
ORDER BY score DESC;
```

## ~ Noise Operator

The tilde makes a match lower relevance rather than disqualifying it:

```sql
-- Include results with 'mysql', treat 'legacy' as a noise term
SELECT id, title,
    MATCH(title, body) AGAINST ('+mysql ~legacy' IN BOOLEAN MODE) AS score
FROM articles
WHERE MATCH(title, body) AGAINST ('+mysql ~legacy' IN BOOLEAN MODE)
ORDER BY score DESC;
```

## * Prefix Wildcard

The `*` must appear at the end of a word, not the beginning:

```sql
-- Matches 'index', 'indexing', 'indexed', 'indexes'
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('index*' IN BOOLEAN MODE);
```

## Phrase Matching with Double Quotes

```sql
-- Exact phrase match
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('"full text search"' IN BOOLEAN MODE);
```

## Grouping with Parentheses

Group terms to create OR conditions within a required set:

```sql
-- Must contain 'mysql', and must contain either 'replication' or 'cluster'
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST (
    '+mysql +(replication cluster)'
    IN BOOLEAN MODE
);
```

## Complex Combined Query

```sql
-- Professional search: must have 'mysql', should prefer 'performance',
-- exclude 'deprecated', allow 'legacy' with lower weight,
-- match any 'partition*' prefix word
SELECT
    id,
    title,
    MATCH(title, body) AGAINST (
        '+mysql >performance -deprecated ~legacy partition*'
        IN BOOLEAN MODE
    ) AS relevance
FROM articles
WHERE MATCH(title, body) AGAINST (
    '+mysql >performance -deprecated ~legacy partition*'
    IN BOOLEAN MODE
)
ORDER BY relevance DESC;
```

## Escaping Special Characters

If your search terms contain special characters, escape them:

```sql
-- Search for a literal plus sign in text
SELECT * FROM articles
WHERE MATCH(title, body) AGAINST ('+"c++" +tutorial' IN BOOLEAN MODE);
```

## Summary

MySQL Boolean mode full-text search operators give you SQL-level control over text search that rivals dedicated search engines for common use cases. Mastering `+`, `-`, `*`, `>`, `<`, `~`, `"..."`, and `()` enables you to build precise, relevance-aware search queries directly within MySQL without external search infrastructure.
