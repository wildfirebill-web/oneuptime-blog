# How to Use Boolean Operators in MySQL Full-Text Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Full-Text Search, Boolean Mode, Search

Description: Learn how to use Boolean operators (+, -, *, ~) in MySQL full-text search to build precise search queries with required, excluded, and weighted terms.

---

## What Is Boolean Mode Full-Text Search

MySQL full-text search supports two modes:
- **Natural Language Mode** - ranks results by relevance score
- **Boolean Mode** - lets you specify required, excluded, and optional terms using operators

Boolean mode gives you fine-grained control over search results.

## Prerequisites

A FULLTEXT index must exist on the searched columns:

```sql
ALTER TABLE articles ADD FULLTEXT INDEX ft_idx (title, body);
```

## Boolean Mode Syntax

```sql
SELECT * FROM articles
WHERE MATCH(title, body) AGAINST ('search expression' IN BOOLEAN MODE);
```

## Operators

| Operator | Meaning | Example |
|---|---|---|
| `+` | Word must be present | `+mysql` |
| `-` | Word must NOT be present | `-oracle` |
| `*` | Wildcard (suffix match) | `data*` |
| `~` | Reduce rank if present | `~performance` |
| `"..."` | Exact phrase | `"query optimizer"` |
| `>` | Increase relevance weight | `>important` |
| `<` | Decrease relevance weight | `<optional` |
| `(...)` | Group subexpressions | `+(mysql +(index query))` |

## Example Queries

### Required and Excluded Terms

Find articles containing "MySQL" but not "Oracle":

```sql
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('+mysql -oracle' IN BOOLEAN MODE);
```

### Exact Phrase Search

Find articles with the exact phrase "query optimizer":

```sql
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('"query optimizer"' IN BOOLEAN MODE);
```

### Wildcard Search

Find articles containing words starting with "replicat":

```sql
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('replicat*' IN BOOLEAN MODE);
```

This matches "replication", "replicate", "replicating", etc.

### Relevance Weighting

Boost articles mentioning "performance" but still show those that don't:

```sql
SELECT id, title,
       MATCH(title, body) AGAINST ('>performance mysql' IN BOOLEAN MODE) AS score
FROM articles
WHERE MATCH(title, body) AGAINST ('>performance mysql' IN BOOLEAN MODE)
ORDER BY score DESC;
```

### Complex Boolean Expression

Find articles about MySQL indexing but not about MyISAM:

```sql
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST (
  '+mysql +(index* partition*) -myisam'
  IN BOOLEAN MODE
);
```

### Phrase with Required Term

Find articles with the phrase "InnoDB buffer pool" that also mention "performance":

```sql
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST (
  '+"InnoDB buffer pool" +performance'
  IN BOOLEAN MODE
);
```

## Getting Relevance Scores

Boolean mode returns a relevance score you can use for sorting:

```sql
SELECT
  id,
  title,
  MATCH(title, body) AGAINST ('+mysql +innodb' IN BOOLEAN MODE) AS relevance
FROM articles
WHERE MATCH(title, body) AGAINST ('+mysql +innodb' IN BOOLEAN MODE)
ORDER BY relevance DESC
LIMIT 10;
```

## Minimum Word Length

MySQL ignores words shorter than `ft_min_word_len` (default: 4 for MyISAM, `innodb_ft_min_token_size` default: 3 for InnoDB):

```sql
SHOW VARIABLES LIKE 'innodb_ft_min_token_size';
```

## Stopwords

Common words like "the", "and", "of" are ignored by default. Check the stopword list:

```sql
SELECT * FROM information_schema.INNODB_FT_DEFAULT_STOPWORD;
```

## Summary

MySQL Boolean mode full-text search uses `+` for required terms, `-` for excluded terms, `*` as a wildcard suffix, and `"..."` for exact phrases. Combine these operators to build precise search queries. Use `>` and `<` to adjust term weights, and retrieve relevance scores to sort results by match quality.
