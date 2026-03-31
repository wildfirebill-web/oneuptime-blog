# How to Use Full-Text Search with CJK Text in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Full-Text Search, CJK, InnoDB, ngram

Description: Learn how to enable full-text search for Chinese, Japanese, and Korean text in MySQL using the built-in ngram full-text parser.

---

## The Challenge with CJK Text

Standard MySQL full-text search tokenizes text by whitespace and punctuation. Chinese, Japanese, and Korean (CJK) writing systems do not use spaces between words, so the default parser produces single huge tokens that cannot be meaningfully searched.

MySQL's built-in **ngram full-text parser** solves this by splitting text into overlapping character sequences of a fixed length, making CJK text searchable without word boundaries.

## How the ngram Parser Works

The ngram parser generates all overlapping substrings of length `n` from the input. For `n=2` (bigrams), the string "数据库" produces tokens: "数据", "据库". A query for "数据" matches any document containing that two-character sequence.

## Enabling the ngram Parser

Specify `WITH PARSER ngram` when creating a full-text index:

```sql
CREATE TABLE cjk_articles (
  id INT PRIMARY KEY AUTO_INCREMENT,
  title VARCHAR(255) NOT NULL,
  body TEXT,
  FULLTEXT INDEX ft_cjk (title, body) WITH PARSER ngram
) ENGINE = InnoDB CHARACTER SET utf8mb4;
```

Or add to an existing table:

```sql
ALTER TABLE cjk_articles
  ADD FULLTEXT INDEX ft_cjk (title, body) WITH PARSER ngram;
```

## Configuring the ngram Token Size

The `ngram_token_size` variable (default `2`) controls the n-gram length. It is read-only at runtime and must be set in `my.cnf`:

```text
[mysqld]
ngram_token_size = 2
```

A token size of 2 works well for Chinese and Japanese. Korean can benefit from size 2 or 3 depending on the vocabulary. After changing this setting, restart MySQL and rebuild all ngram full-text indexes.

```bash
sudo systemctl restart mysql
```

## Inserting and Searching CJK Text

```sql
INSERT INTO cjk_articles (title, body) VALUES
  ('MySQL数据库优化', '如何使用索引提高查询性能'),
  ('データベース設計', 'インデックスを使った高速化の方法'),
  ('데이터베이스 인덱스', '쿼리 성능을 향상시키는 방법');

-- Search in Natural Language Mode
SELECT id, title,
  MATCH(title, body) AGAINST('数据库' IN NATURAL LANGUAGE MODE) AS score
FROM cjk_articles
WHERE MATCH(title, body) AGAINST('数据库' IN NATURAL LANGUAGE MODE)
ORDER BY score DESC;
```

## Boolean Mode Searches

Boolean mode with ngram supports the same operators as with the default parser:

```sql
-- Require '数据库' and optionally include '优化'
SELECT title FROM cjk_articles
WHERE MATCH(title, body)
  AGAINST('+数据库 优化' IN BOOLEAN MODE);
```

## Mixed CJK and Latin Text

A table can have both CJK and Latin text. The ngram parser handles Latin words by also splitting them into n-grams, which means Latin searches behave differently - "mysql" with `ngram_token_size=2` becomes tokens "my", "ys", "sq", "ql":

```sql
-- Latin text searched as bigrams
SELECT * FROM cjk_articles
WHERE MATCH(title, body) AGAINST('ql' IN BOOLEAN MODE);
```

For tables containing both languages, consider separate indexes: one with the default parser for Latin text and one with `WITH PARSER ngram` for CJK fields.

## Checking ngram Token Size

```sql
SHOW VARIABLES LIKE 'ngram_token_size';
```

## Summary

CJK full-text search in MySQL requires the ngram parser, which generates overlapping character substrings instead of whitespace-delimited tokens. Create full-text indexes with `WITH PARSER ngram`, configure the token size via `ngram_token_size` in `my.cnf`, and use `utf8mb4` character encoding. After any configuration change, restart MySQL and rebuild all ngram full-text indexes.
