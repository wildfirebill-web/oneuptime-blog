# How to Fix "Code: 60 Table doesn't exist" in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Error, Table, Troubleshooting, DDL

Description: Resolve ClickHouse error Code 60 "Table doesn't exist" by identifying missing tables, detached states, and database context issues.

---

ClickHouse error `Code: 60. DB::Exception: Table <db>.<table> doesn't exist` means the query references a table that ClickHouse cannot find in its metadata. This can happen due to a dropped table, a detached table, a wrong database context, or a case sensitivity mismatch.

## Verify the Table Exists

```sql
-- List all tables in a database
SHOW TABLES FROM my_database;

-- Check system.tables
SELECT database, name, engine
FROM system.tables
WHERE name LIKE '%my_table%';
```

## Check the Correct Database

If you are connected without specifying a database, ClickHouse uses the `default` database:

```sql
SELECT currentDatabase();
```

Switch to the right database:

```sql
USE my_database;
-- or qualify the table name
SELECT * FROM my_database.my_table LIMIT 5;
```

## Check for Detached Tables

Tables that have been detached are not visible via `SHOW TABLES` but their data remains on disk:

```sql
SELECT database, name
FROM system.detached_tables
WHERE name = 'my_table';
```

Reattach the table:

```sql
ATTACH TABLE my_database.my_table;
```

## Recover a Dropped Table

If the table was dropped, check if data files still exist on disk:

```bash
ls /var/lib/clickhouse/data/my_database/my_table/
```

If metadata was lost but data remains, recreate the table with the original DDL and then run:

```sql
ATTACH TABLE my_database.my_table;
```

## Case Sensitivity

ClickHouse table names are case-sensitive on Linux. `MyTable` and `mytable` are different tables:

```sql
SELECT name FROM system.tables
WHERE lower(name) = 'my_table' AND database = 'my_database';
```

## Check Distributed Tables

For distributed queries, ensure all shards have the local table:

```sql
SELECT hostName(), database, name
FROM clusterAllReplicas('my_cluster', system.tables)
WHERE name = 'my_table';
```

## Summary

Error Code 60 in ClickHouse points to a missing table reference. Start by confirming the table exists with the correct database prefix and casing. Check `system.detached_tables` for detached tables and use `ATTACH TABLE` to restore them. If data files remain on disk after an accidental drop, recreate the table schema and reattach to recover without data loss.
