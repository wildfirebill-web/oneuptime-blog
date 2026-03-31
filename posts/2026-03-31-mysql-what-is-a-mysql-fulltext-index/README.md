# What Is a MySQL FULLTEXT Index

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, FULLTEXT Index, Full-Text Search, InnoDB, Search

Description: Learn what a MySQL FULLTEXT index is, how it enables natural language and boolean search, and how to create and use one effectively.

---

## What Is a FULLTEXT Index

A FULLTEXT index is a special type of MySQL index designed for full-text search on character-based column data. Unlike a standard B-tree index that matches exact values, a FULLTEXT index tokenizes text into individual words (called tokens) and builds an inverted index that maps each word to the rows containing it.

FULLTEXT indexes are supported on `CHAR`, `VARCHAR`, and `TEXT` columns. Since MySQL 5.6, they are supported by InnoDB tables.

## Creating a FULLTEXT Index

Create a FULLTEXT index at table creation time:

```sql
CREATE TABLE articles (
  id INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(200),
  body TEXT,
  FULLTEXT INDEX idx_fulltext (title, body)
);
```

Or add one to an existing table:

```sql
ALTER TABLE articles ADD FULLTEXT INDEX idx_fulltext (title, body);
```

You can also create it with CREATE INDEX syntax:

```sql
CREATE FULLTEXT INDEX idx_body ON articles (body);
```

## Natural Language Mode Search

The default search mode is natural language, which ranks results by relevance:

```sql
SELECT id, title,
  MATCH(title, body) AGAINST ('database performance') AS relevance
FROM articles
WHERE MATCH(title, body) AGAINST ('database performance');
```

MySQL calculates a relevance score for each matching row. Rows with higher scores contain the search terms more frequently or in more significant positions. Results are automatically sorted by relevance when no ORDER BY is specified.

## Boolean Mode Search

Boolean mode gives you more control over the search with operators:

```sql
-- Must contain 'mysql', must not contain 'oracle'
SELECT id, title FROM articles
WHERE MATCH(title, body) AGAINST ('+mysql -oracle' IN BOOLEAN MODE);

-- Must contain 'mysql', optionally contains 'performance'
SELECT id, title FROM articles
WHERE MATCH(title, body) AGAINST ('+mysql performance' IN BOOLEAN MODE);

-- Wildcard search - words starting with 'perform'
SELECT id, title FROM articles
WHERE MATCH(title, body) AGAINST ('perform*' IN BOOLEAN MODE);

-- Phrase search
SELECT id, title FROM articles
WHERE MATCH(title, body) AGAINST ('"query optimizer"' IN BOOLEAN MODE);
```

Boolean mode operators:
- `+` - the word must appear
- `-` - the word must not appear
- `*` - wildcard (append to a word)
- `"..."` - phrase match

## Query Expansion Mode

Query expansion mode performs a two-pass search. The first pass finds the best matching rows, then MySQL expands the query with additional terms from those rows and searches again:

```sql
SELECT id, title FROM articles
WHERE MATCH(title, body) AGAINST ('database' WITH QUERY EXPANSION);
```

This returns a broader set of results that may be topically related even if they do not contain the exact search term.

## Minimum Word Length

By default, MySQL ignores words shorter than the minimum token length. For InnoDB, the default minimum is 3 characters:

```sql
SHOW VARIABLES LIKE 'innodb_ft_min_token_size';
```

Change it in `my.cnf` and rebuild the index:

```ini
[mysqld]
innodb_ft_min_token_size = 2
```

```bash
sudo systemctl restart mysql
```

```sql
-- Rebuild the FULLTEXT index after changing token size
ALTER TABLE articles DROP INDEX idx_fulltext;
ALTER TABLE articles ADD FULLTEXT INDEX idx_fulltext (title, body);
```

## Stopwords

MySQL has a built-in stopword list that excludes common words like "the", "and", "or" from indexing. You can customize this for InnoDB:

```sql
-- View current stopwords
SELECT * FROM information_schema.INNODB_FT_DEFAULT_STOPWORD;

-- Use a custom stopword table
CREATE TABLE my_stopwords (value VARCHAR(30));
INSERT INTO my_stopwords VALUES ('the'), ('and'), ('is');

SET GLOBAL innodb_ft_server_stopword_table = 'mydb/my_stopwords';
```

## Checking the FULLTEXT Index

View the indexed words (for debugging):

```sql
SET GLOBAL innodb_ft_aux_table = 'mydb/articles';
SELECT * FROM information_schema.INNODB_FT_INDEX_TABLE LIMIT 20;
```

## Limitations

- FULLTEXT indexes only work on character columns (`CHAR`, `VARCHAR`, `TEXT`)
- All columns in a MATCH() call must be from the same FULLTEXT index
- FULLTEXT search is not effective on very small tables (fewer than a few hundred rows)
- MySQL's built-in FULLTEXT search is less powerful than dedicated search engines like Elasticsearch or Solr

## Summary

A MySQL FULLTEXT index builds an inverted word index on character columns, enabling fast full-text search with natural language ranking, boolean operators, and phrase matching. It is a good solution for application-level search on medium-scale datasets. For large-scale search requirements with advanced relevance tuning, consider a dedicated search engine alongside MySQL.
