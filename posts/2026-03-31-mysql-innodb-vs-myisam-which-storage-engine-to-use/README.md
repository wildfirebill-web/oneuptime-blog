# MySQL InnoDB vs MyISAM: Which Storage Engine to Use

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB, MyISAM

Description: Compare MySQL's InnoDB and MyISAM storage engines on transactions, locking, crash recovery, and performance to make the right choice for your workload.

---

MySQL supports multiple storage engines, with InnoDB and MyISAM being the two most historically prominent. Since MySQL 5.5, InnoDB has been the default and is the right choice for virtually all new workloads. However, understanding why requires knowing how they differ.

## Transaction Support

The most critical difference: InnoDB supports ACID transactions. MyISAM does not.

```sql
-- InnoDB: transactions work correctly
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- MyISAM: no transaction support - partial writes cannot be rolled back
-- BEGIN/COMMIT are silently ignored
```

## Locking Granularity

InnoDB uses row-level locking. MyISAM uses table-level locking. This makes a significant difference under concurrent write workloads.

```sql
-- Check which engine a table uses
SHOW TABLE STATUS LIKE 'orders'\G
-- Look for Engine: InnoDB or Engine: MyISAM

-- Convert a MyISAM table to InnoDB
ALTER TABLE legacy_logs ENGINE = InnoDB;
```

With MyISAM, a single write query locks the entire table, blocking all other reads and writes. InnoDB only locks the affected rows.

## Foreign Keys

InnoDB enforces foreign key constraints. MyISAM accepts the `FOREIGN KEY` syntax but silently ignores it.

```sql
-- InnoDB enforces referential integrity
CREATE TABLE orders (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB;
```

## Crash Recovery

InnoDB maintains a transaction log (redo log) and supports automatic crash recovery. If MySQL crashes mid-write, InnoDB rolls back incomplete transactions on restart. MyISAM has no crash recovery - a corrupted MyISAM table requires manual repair.

```sql
-- Repair a crashed MyISAM table (not needed with InnoDB)
REPAIR TABLE legacy_table;
```

## Full-Text Search

Historically, MyISAM was the only engine supporting FULLTEXT indexes. Since MySQL 5.6, InnoDB also supports FULLTEXT indexes, removing one of the last reasons to use MyISAM.

```sql
-- InnoDB FULLTEXT index (MySQL 5.6+)
ALTER TABLE articles ADD FULLTEXT INDEX ft_body (body);
SELECT * FROM articles WHERE MATCH(body) AGAINST('database performance' IN NATURAL LANGUAGE MODE);
```

## Performance Characteristics

MyISAM can be faster for read-heavy workloads with no concurrent writes because it avoids transaction overhead. InnoDB's buffer pool caches both data and indexes, making it competitive for most read workloads.

| Feature | InnoDB | MyISAM |
|---|---|---|
| Transactions | Yes | No |
| Row-level locking | Yes | No (table-level) |
| Foreign keys | Yes | No |
| Crash recovery | Automatic | Manual REPAIR |
| FULLTEXT index | Yes (5.6+) | Yes |
| Write concurrency | High | Low |

## Summary

Use InnoDB for all production workloads - it is the MySQL default for good reason. It provides transactions, row-level locking, foreign key enforcement, and automatic crash recovery. MyISAM is a legacy engine appropriate only for read-only archive tables or when working with very old MySQL versions. Any existing MyISAM tables should be migrated to InnoDB using `ALTER TABLE ... ENGINE=InnoDB`.
