# How MySQL Full-Text Search Works Internally

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Full-Text Search, InnoDB, FULLTEXT Index, Performance

Description: Understand how MySQL InnoDB full-text search builds inverted indexes, processes queries with natural language and boolean modes, and ranks results.

---

## What Is MySQL Full-Text Search

MySQL Full-Text Search (FTS) uses inverted indexes to enable fast text searching across large text columns. Unlike LIKE queries, FTS breaks text into tokens, builds an inverted index (token -> list of documents), and can rank results by relevance.

InnoDB supports full-text indexes since MySQL 5.6.

## How the Inverted Index Works

When you create a FULLTEXT index, MySQL:
1. Tokenizes the text in the indexed columns using a parser.
2. Removes stopwords (common words like "the", "is", "a").
3. Stores the remaining tokens in an inverted index: for each token, records which rows contain it and how many times.

The inverted index is stored in a set of special auxiliary tables within the InnoDB tablespace.

## Creating a FULLTEXT Index

```sql
CREATE TABLE articles (
  id INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(200) NOT NULL,
  body TEXT NOT NULL,
  FULLTEXT idx_ft (title, body)
);

-- Or add to existing table:
ALTER TABLE articles ADD FULLTEXT INDEX idx_ft (title, body);
```

## Natural Language Mode (Default)

Natural language mode ranks results by relevance (TF-IDF based score):

```sql
SELECT id, title,
  MATCH(title, body) AGAINST ('MySQL performance tuning') AS relevance_score
FROM articles
WHERE MATCH(title, body) AGAINST ('MySQL performance tuning')
ORDER BY relevance_score DESC
LIMIT 10;
```

MySQL calculates relevance as a function of:
- Term frequency (TF) - how often the word appears in this document.
- Inverse document frequency (IDF) - how rare the word is across all documents.

Words that appear in more than 50% of rows are treated as too common and ignored.

## Boolean Mode

Boolean mode supports explicit operators and does not rank by relevance by default:

```sql
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('+MySQL +performance -Oracle' IN BOOLEAN MODE);
```

Boolean operators:
- `+word` - word must be present.
- `-word` - word must not be present.
- `word*` - prefix wildcard.
- `"exact phrase"` - must match as a phrase.
- `>word` - increase word's contribution to relevance.
- `<word` - decrease word's contribution.
- `(group)` - grouping with parentheses.

```sql
-- Find articles about MySQL indexing but not about full-text
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('+MySQL +index* -"full-text"' IN BOOLEAN MODE);
```

## Query Expansion Mode

Query expansion performs the search twice - first normally, then expands with top result words:

```sql
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('MySQL' WITH QUERY EXPANSION)
ORDER BY MATCH(title, body) AGAINST ('MySQL' WITH QUERY EXPANSION) DESC
LIMIT 10;
```

This helps find related documents even if they don't contain the exact search term.

## Minimum Word Length

By default, InnoDB ignores words shorter than `innodb_ft_min_token_size` characters:

```sql
SHOW VARIABLES LIKE 'innodb_ft_min_token_size';
-- Default: 3
```

To index 2-character words:

```text
[mysqld]
innodb_ft_min_token_size = 2
```

After changing, rebuild the index:

```sql
OPTIMIZE TABLE articles;
```

## Custom Stopwords

```sql
-- View current stopwords
SELECT * FROM INFORMATION_SCHEMA.INNODB_FT_DEFAULT_STOPWORD;

-- Create custom stopword table
CREATE TABLE my_stopwords (value VARCHAR(30));
INSERT INTO my_stopwords VALUES ('example'), ('test');

SET GLOBAL innodb_ft_server_stopword_table = 'myapp/my_stopwords';

-- Rebuild the index to apply new stopwords
ALTER TABLE articles DROP INDEX idx_ft;
ALTER TABLE articles ADD FULLTEXT INDEX idx_ft (title, body);
```

## N-gram Parser for CJK and Mid-Word Search

The built-in parser splits on spaces and punctuation (word boundaries). For Chinese, Japanese, Korean (CJK) text, or to support mid-word search, use the ngram parser:

```sql
CREATE TABLE products (
  id INT PRIMARY KEY,
  name VARCHAR(200),
  FULLTEXT INDEX ft_ngram (name) WITH PARSER ngram
);
```

```sql
-- N-gram can match substrings
SELECT * FROM products
WHERE MATCH(name) AGAINST ('oard' IN BOOLEAN MODE);
-- Matches "Keyboard", "Clipboard", etc.
```

## Monitoring Full-Text Index Size

```sql
SELECT
  INDEX_NAME,
  stat_value AS pages
FROM mysql.innodb_index_stats
WHERE database_name = 'myapp'
  AND table_name = 'articles'
  AND stat_name = 'size';
```

## Summary

MySQL InnoDB full-text search uses inverted indexes to tokenize text, remove stopwords, and enable relevance-ranked queries with natural language mode or operator-rich boolean mode. Natural language mode ranks results using TF-IDF scoring; boolean mode gives precise control over required and excluded terms. For substring and CJK text matching, use the ngram parser. Tune `innodb_ft_min_token_size` and custom stopwords to control what gets indexed.
