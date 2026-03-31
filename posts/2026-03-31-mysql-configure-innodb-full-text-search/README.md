# How to Configure InnoDB Full-Text Search in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Full-Text Search, InnoDB, FULLTEXT, Configuration

Description: Configure and tune InnoDB FULLTEXT indexes in MySQL including minimum word length, stopwords, boolean mode, and performance settings.

---

## Creating a FULLTEXT Index

InnoDB supports full-text search on CHAR, VARCHAR, and TEXT columns. Add a FULLTEXT index when creating the table or afterward:

```sql
-- At table creation
CREATE TABLE articles (
  id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  title   VARCHAR(255) NOT NULL,
  body    TEXT NOT NULL,
  FULLTEXT idx_ft (title, body)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- On an existing table
ALTER TABLE articles ADD FULLTEXT INDEX idx_ft (title, body);

-- Or use CREATE FULLTEXT INDEX syntax
CREATE FULLTEXT INDEX idx_ft ON articles (title, body);
```

## Querying with MATCH ... AGAINST

```sql
-- Natural language mode (default): ranks results by relevance
SELECT id, title,
       MATCH(title, body) AGAINST ('database performance tuning') AS score
FROM articles
WHERE MATCH(title, body) AGAINST ('database performance tuning')
ORDER BY score DESC
LIMIT 10;

-- Boolean mode: supports +/- operators, wildcards, and phrases
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST (
  '+database +performance -deprecated "full text"*' IN BOOLEAN MODE
);

-- Query expansion: broadens search using top result terms
SELECT id, title
FROM articles
WHERE MATCH(title, body)
  AGAINST ('InnoDB' WITH QUERY EXPANSION);
```

## Tuning Minimum Word Length

By default, InnoDB indexes words of 3 characters or more. Short words like "Go", "AI", or "VM" are ignored.

```sql
-- Check current setting
SHOW VARIABLES LIKE 'innodb_ft_min_token_size';

-- In my.cnf to index 2-character words
-- [mysqld]
-- innodb_ft_min_token_size = 2
```

After changing `innodb_ft_min_token_size`, you must rebuild all FULLTEXT indexes:

```bash
# Rebuild all full-text indexes in a database
mysql -u root -p mydb -e "
  REPAIR TABLE articles QUICK;
"
```

## Configuring Stopwords

Stopwords are common words excluded from indexing. InnoDB has a built-in stopword list. You can replace it with a custom list:

```sql
-- Create a custom stopword table
CREATE TABLE my_stopwords (value VARCHAR(30)) ENGINE=InnoDB;
INSERT INTO my_stopwords VALUES ('the'), ('is'), ('at'), ('your_domain_specific_word');

-- Tell InnoDB to use it (restart required for server-level)
-- In my.cnf:
-- innodb_ft_server_stopword_table = mydb/my_stopwords

-- Or set per-table before creating the index
SET GLOBAL innodb_ft_server_stopword_table = 'mydb/my_stopwords';
```

Disable stopwords entirely for exact matching:

```sql
SET GLOBAL innodb_ft_enable_stopword = OFF;
```

## Monitoring FULLTEXT Index Performance

```sql
-- View internal full-text search tables
SET GLOBAL innodb_ft_aux_table = 'mydb/articles';

-- Index words and their document frequency
SELECT * FROM information_schema.innodb_ft_index_cache LIMIT 20;
SELECT * FROM information_schema.innodb_ft_index_table LIMIT 20;

-- Deleted records pending purge
SELECT * FROM information_schema.innodb_ft_deleted;
SELECT * FROM information_schema.innodb_ft_being_deleted;
```

## Optimizing Full-Text Index Rebuild

After heavy DELETE operations, deleted document IDs accumulate in the FTS index. Optimize the table to rebuild the index and reclaim space:

```sql
-- Rebuilds the FULLTEXT index and removes deleted document entries
OPTIMIZE TABLE articles;
```

For large tables, `OPTIMIZE TABLE` can be slow and locks the table. In MySQL 8, use `ALTER TABLE ... ALGORITHM=INPLACE` or plan for a maintenance window.

## Configuration Reference

Key variables to set in `my.cnf`:

```bash
[mysqld]
innodb_ft_min_token_size     = 3      # minimum word length to index
innodb_ft_max_token_size     = 84     # maximum word length
innodb_ft_enable_stopword    = ON     # enable/disable stopwords
innodb_ft_result_cache_limit = 2000M  # memory for FTS result cache
```

## Summary

InnoDB full-text search requires FULLTEXT indexes, queries using MATCH/AGAINST syntax, and appropriate tuning of `innodb_ft_min_token_size` and stopword lists. For short tokens like product codes or abbreviations, reduce the minimum token size and rebuild indexes. Monitor the internal FTS auxiliary tables to understand index state, and run `OPTIMIZE TABLE` periodically to purge deleted document references and maintain search performance.
