# How to Use OPTIMIZE TABLE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, OPTIMIZE TABLE, InnoDB, Table Maintenance

Description: Learn how to use OPTIMIZE TABLE in MySQL to reclaim disk space, defragment InnoDB tables, and rebuild indexes after large deletes or updates.

---

## What Is OPTIMIZE TABLE

`OPTIMIZE TABLE` reorganizes the physical storage of a table to reduce fragmentation, reclaim unused disk space, and update index statistics. It is useful after:

- Large numbers of `DELETE` or `UPDATE` operations
- Changing many VARCHAR columns to shorter values
- Frequent inserts and deletes that cause page fragmentation

## Syntax

```sql
OPTIMIZE TABLE table_name;
OPTIMIZE TABLE table1, table2, table3;
```

## How It Works by Storage Engine

### InnoDB

InnoDB does not support native `OPTIMIZE TABLE`. Instead, MySQL rebuilds the table using `ALTER TABLE ... ENGINE=InnoDB` internally:

1. Creates a new `.ibd` file
2. Copies all rows in clustered index order
3. Rebuilds all secondary indexes
4. Replaces the old file with the new one

This rebuilds the table in an efficient, unfragmented layout.

### MyISAM

For MyISAM, `OPTIMIZE TABLE`:
1. Fixes any deleted or split rows
2. Rebuilds all indexes
3. Updates index statistics

## Example

```sql
-- Check table before optimizing
SELECT
  table_name,
  data_length / 1024 / 1024      AS data_mb,
  data_free / 1024 / 1024        AS free_mb,
  index_length / 1024 / 1024     AS index_mb
FROM information_schema.tables
WHERE table_schema = 'mydb'
  AND table_name = 'orders';
```

```text
+------------+---------+---------+----------+
| table_name | data_mb | free_mb | index_mb |
+------------+---------+---------+----------+
| orders     | 1024.00 |  256.00 |   512.00 |
+------------+---------+---------+----------+
```

`data_free` shows 256 MB of fragmented, unused space.

```sql
OPTIMIZE TABLE mydb.orders;
```

```text
+-------------+----------+----------+-------------------------------------------------------------------+
| Table       | Op       | Msg_type | Msg_text                                                          |
+-------------+----------+----------+-------------------------------------------------------------------+
| mydb.orders | optimize | note     | Table does not support optimize, doing recreate + analyze instead |
| mydb.orders | optimize | status   | OK                                                                |
+-------------+----------+----------+-------------------------------------------------------------------+
```

The "note" for InnoDB is normal and expected.

## Running OPTIMIZE TABLE Online (No Table Lock)

By default, `OPTIMIZE TABLE` on InnoDB uses Online DDL and does not require a full table lock in MySQL 5.6+. However, for large tables you may want to use `pt-online-schema-change`:

```bash
pt-online-schema-change --alter "ENGINE=InnoDB" \
  --host=localhost --user=root --password=secret \
  D=mydb,t=orders --execute
```

This avoids locking the table during the rebuild.

## Scheduling Regular OPTIMIZE Jobs

Use a cron job or MySQL Event Scheduler:

```sql
CREATE EVENT optimize_orders_weekly
ON SCHEDULE EVERY 1 WEEK
STARTS '2026-04-06 02:00:00'
DO
  OPTIMIZE TABLE mydb.orders;
```

Enable the Event Scheduler first:

```sql
SET GLOBAL event_scheduler = ON;
```

## When OPTIMIZE TABLE Is NOT Needed

- If `data_free` is low, fragmentation is minimal
- For append-only tables (only inserts, no deletes)
- Frequently updated tables where you continuously optimize - overhead may outweigh benefits

## Automating with mysqlcheck

```bash
# Optimize all tables in a database
mysqlcheck -u root -p --optimize mydb

# Optimize all databases
mysqlcheck -u root -p --optimize --all-databases
```

## Summary

`OPTIMIZE TABLE` reclaims fragmented space and rebuilds indexes for MySQL tables. For InnoDB, it performs a full table rebuild. Check `data_free` in `information_schema.tables` to determine if optimization is needed. For large production tables, use `pt-online-schema-change` to avoid downtime during the rebuild.
