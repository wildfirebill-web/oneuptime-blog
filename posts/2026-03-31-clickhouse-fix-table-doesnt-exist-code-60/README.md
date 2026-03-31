# How to Fix 'Code: 60 Table doesn't exist' in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Troubleshooting, Error, Table, Schema

Description: Diagnose and fix the 'Code: 60, e.DB::Exception: Table doesn't exist' error in ClickHouse caused by wrong database context, case sensitivity, or metadata issues.

---

ClickHouse error code 60 ("Table doesn't exist") is a common error that can be confusing because the table appears to exist when you check it. This guide covers all the common causes and their fixes.

## Reproducing the Error

```text
Code: 60. DB::Exception: Table default.my_table doesn't exist. (UNKNOWN_TABLE)
```

## Cause 1: Wrong Database Context

The most common cause - you are connected to a different database than where the table lives.

Check your current database:

```sql
SELECT currentDatabase();
```

List tables in the correct database:

```sql
SHOW TABLES FROM mydb;
```

Fix by using a fully qualified table name:

```sql
SELECT * FROM mydb.my_table LIMIT 10;
```

Or switch your default database:

```bash
clickhouse-client --database mydb --query "SELECT * FROM my_table LIMIT 10"
```

## Cause 2: Case Sensitivity

ClickHouse table names are case-sensitive on Linux:

```sql
-- These are different tables
SELECT * FROM MyTable;
SELECT * FROM mytable;
SELECT * FROM MYTABLE;
```

Check the exact table name:

```sql
SELECT name FROM system.tables WHERE database = 'mydb' AND lower(name) = 'my_table';
```

## Cause 3: Table Dropped or Not Created

Verify the table exists in the system catalog:

```sql
SELECT database, name, engine
FROM system.tables
WHERE name = 'my_table';
```

If it does not appear, the table was either dropped or never created on this server. Check if it exists on another replica:

```sql
-- Check from a different node
SELECT * FROM remote('other-clickhouse-host', system.tables)
WHERE name = 'my_table';
```

## Cause 4: Metadata Inconsistency After Crash

After an unclean shutdown, metadata files may be inconsistent. Check for orphaned or corrupt metadata:

```bash
ls /var/lib/clickhouse/metadata/mydb/
```

Compare with what ClickHouse reports:

```sql
SELECT name FROM system.tables WHERE database = 'mydb';
```

If a `.sql` file exists in the metadata directory but the table does not appear in `system.tables`, restart ClickHouse to reload metadata:

```bash
sudo systemctl restart clickhouse-server
```

## Cause 5: Distributed Table Pointing to Wrong Table

If you are using a Distributed table engine, the underlying table must exist on the shard:

```sql
-- Check what the Distributed table points to
SHOW CREATE TABLE mydb.distributed_table;
```

The `local_table_name` parameter must match an existing MergeTree table on each shard.

## Cause 6: Table Engine is a View on a Missing Source

```sql
-- Check if the table is a view
SELECT name, engine, create_table_query
FROM system.tables
WHERE name = 'my_table';
```

If it is a view pointing to a dropped base table, recreate the base table or drop and recreate the view.

## Summary

"Code: 60 Table doesn't exist" errors are usually caused by querying the wrong database context, case sensitivity mismatches in table names, or the table genuinely not existing on the current node. Use `system.tables` to verify existence, always use fully qualified `database.table` names, and check for metadata inconsistencies after unclean shutdowns.
