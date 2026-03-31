# How to Use LINEAR KEY Partitioning in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Partition, Linear Key, Performance, InnoDB

Description: Learn how to use LINEAR KEY partitioning in MySQL, how it compares to regular KEY partitioning, and when it offers advantages for dynamic partition management.

---

## What Is LINEAR KEY Partitioning?

LINEAR KEY partitioning is the linear-algorithm variant of KEY partitioning in MySQL. Like LINEAR HASH, it uses a powers-of-two algorithm internally instead of a simple modulus operation. This makes adding and removing partitions faster because fewer rows need to be moved when changing the partition count.

KEY partitioning itself differs from HASH in that MySQL internally hashes the column value using its own hashing function rather than a user-supplied expression. LINEAR KEY extends this with the linear algorithm.

## Basic Syntax

```sql
CREATE TABLE sessions (
    session_id VARCHAR(64) NOT NULL,
    user_id INT NOT NULL,
    ip_address VARCHAR(45),
    created_at DATETIME,
    PRIMARY KEY (session_id)
)
PARTITION BY LINEAR KEY (session_id)
PARTITIONS 8;
```

## LINEAR KEY with Multiple Columns

```sql
CREATE TABLE access_logs (
    log_id BIGINT NOT NULL AUTO_INCREMENT,
    server_id INT NOT NULL,
    endpoint VARCHAR(255),
    status_code SMALLINT,
    logged_at DATETIME NOT NULL,
    PRIMARY KEY (log_id, server_id)
)
PARTITION BY LINEAR KEY (server_id)
PARTITIONS 16;
```

## Using the Primary Key Implicitly

If you omit the column list from `LINEAR KEY ()`, MySQL uses all columns in the primary key:

```sql
CREATE TABLE cache_entries (
    cache_key VARCHAR(128) NOT NULL,
    cache_value MEDIUMBLOB,
    expires_at DATETIME,
    PRIMARY KEY (cache_key)
)
PARTITION BY LINEAR KEY()
PARTITIONS 4;
```

## Differences: LINEAR KEY vs KEY vs LINEAR HASH

```text
KEY:         Uses MySQL internal hash; even distribution; slower to add partitions
LINEAR KEY:  Same internal hash; less uniform distribution; faster partition management
HASH:        User-defined expression; even distribution; slower to add partitions
LINEAR HASH: User-defined expression; less uniform; faster partition management
```

## Adding Partitions to a LINEAR KEY Table

One of the main benefits of LINEAR KEY is fast partition expansion:

```sql
ALTER TABLE access_logs
ADD PARTITION PARTITIONS 8;
-- Grows partition count with less data movement
```

## Checking Partition Metadata

```sql
SELECT
    PARTITION_NAME,
    PARTITION_METHOD,
    TABLE_ROWS,
    DATA_LENGTH
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'access_logs'
ORDER BY PARTITION_ORDINAL_POSITION;
```

## Reducing Partitions

To reduce the number of partitions:

```sql
ALTER TABLE access_logs
COALESCE PARTITION 4;
```

## When to Choose LINEAR KEY

- You are partitioning on non-integer columns like `VARCHAR` where KEY is required
- Partition count needs to grow dynamically over time
- You cannot define a clean numeric expression for HASH partitioning
- Faster management outweighs the cost of slightly uneven distribution

## Summary

LINEAR KEY partitioning combines the flexibility of MySQL's internal key-based hashing with the linear algorithm that makes partition expansion and contraction faster. It is a strong choice for large tables partitioned on string or composite keys where the partition schema needs to evolve without long maintenance windows.
