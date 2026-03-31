# How to Configure innodb_ft_min_token_size in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Full-Text Search, InnoDB, Configuration, Index

Description: Learn how to configure innodb_ft_min_token_size in MySQL to control the minimum word length indexed in InnoDB full-text searches.

---

## What is innodb_ft_min_token_size?

The `innodb_ft_min_token_size` system variable controls the minimum length of words that InnoDB indexes for full-text search. Words shorter than this threshold are silently excluded from the full-text index. The default value is `3`, meaning two-letter words like "to", "is", and "in" are not indexed.

This setting directly affects which search terms can match results. If a user searches for "AI" or "go" and the minimum token size is 3, no results are returned even if those terms appear in your data.

## Checking the Current Value

```sql
SHOW VARIABLES LIKE 'innodb_ft_min_token_size';
```

```text
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| innodb_ft_min_token_size | 3     |
+--------------------------+-------+
```

## Changing innodb_ft_min_token_size

This variable is read-only at runtime and must be set in the MySQL configuration file. Edit `/etc/mysql/my.cnf` or `/etc/my.cnf`:

```text
[mysqld]
innodb_ft_min_token_size = 2
```

After saving, restart MySQL:

```bash
sudo systemctl restart mysql
```

Setting the value to `2` allows two-character words to be indexed. Setting it to `1` allows single-character indexing, which can significantly increase index size.

## Rebuilding the Full-Text Index

After changing `innodb_ft_min_token_size`, you must rebuild all existing full-text indexes for the change to take effect. Simply restarting MySQL does not reindex existing data.

```sql
-- Drop and recreate the full-text index
ALTER TABLE articles DROP INDEX ft_idx;
ALTER TABLE articles ADD FULLTEXT INDEX ft_idx (title, body);
```

Alternatively, use `OPTIMIZE TABLE` to rebuild the index in place:

```sql
OPTIMIZE TABLE articles;
```

## Practical Example

Consider a products table where users search for short technical terms like "AI", "ML", or "Go":

```sql
CREATE TABLE products (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(255),
  description TEXT,
  FULLTEXT INDEX ft_desc (name, description)
);

INSERT INTO products (name, description) VALUES
  ('AI Toolkit', 'Build ML models with Python'),
  ('Go SDK', 'Write services in Go language');

-- With default min_token_size=3, this returns nothing
SELECT * FROM products
WHERE MATCH(name, description) AGAINST('AI' IN BOOLEAN MODE);
```

After setting `innodb_ft_min_token_size = 2` and rebuilding the index:

```sql
-- Now 'AI' and 'Go' are indexed and match correctly
SELECT * FROM products
WHERE MATCH(name, description) AGAINST('AI' IN BOOLEAN MODE);
```

## Companion Variable: innodb_ft_max_token_size

There is a companion variable `innodb_ft_max_token_size` (default `84`) that caps the maximum indexed word length. Words longer than this are also excluded. For most workloads the default maximum is sufficient, but very technical documentation with long identifiers may require tuning.

```sql
SHOW VARIABLES LIKE 'innodb_ft_%token_size';
```

## Performance Considerations

Lowering `innodb_ft_min_token_size` increases the number of tokens stored in the full-text index, which can:

- Increase index size on disk
- Slow down index builds and OPTIMIZE TABLE operations
- Slightly increase memory usage during searches

For most OLTP applications a value of `2` is a reasonable compromise. Avoid setting it to `1` unless your search use case genuinely requires single-character indexing.

## Summary

`innodb_ft_min_token_size` controls the minimum word length for InnoDB full-text indexes. The default of `3` excludes short but meaningful words. To index shorter terms, set the variable in `my.cnf`, restart MySQL, and rebuild affected full-text indexes using `ALTER TABLE` or `OPTIMIZE TABLE`.
