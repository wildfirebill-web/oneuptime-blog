# How to Use Replication Filters in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Replication, Replication Filters, Database Administration

Description: Learn how to configure MySQL replication filters to selectively replicate specific databases or tables from primary to replica.

---

## What Are Replication Filters

Replication filters let you control which databases or tables are replicated. Filters can be applied on the primary (binary log filters) or on the replica (relay log filters).

Common use cases:
- Exclude system or audit tables from replication
- Replicate only production databases to a reporting replica
- Avoid replicating temporary or cache tables

## Types of Filters

### Primary-Side Filters (Binary Log)

These control what the primary writes to the binary log:

| Variable | Effect |
|---|---|
| `binlog_do_db` | Only log changes for this database |
| `binlog_ignore_db` | Do not log changes for this database |

### Replica-Side Filters (Relay Log)

These control what the replica applies from the relay log:

| Variable | Effect |
|---|---|
| `replicate_do_db` | Only replicate this database |
| `replicate_ignore_db` | Skip this database |
| `replicate_do_table` | Only replicate this table |
| `replicate_ignore_table` | Skip this table |
| `replicate_wild_do_table` | Replicate tables matching a pattern |
| `replicate_wild_ignore_table` | Ignore tables matching a pattern |

## Configure Filters in my.cnf

### Example: Replicate Only Specific Databases

```ini
[mysqld]
replicate_do_db = production
replicate_do_db = inventory
```

### Example: Ignore Specific Databases

```ini
[mysqld]
replicate_ignore_db = test
replicate_ignore_db = temp_work
```

### Example: Ignore Specific Tables

```ini
[mysqld]
replicate_ignore_table = production.audit_log
replicate_ignore_table = production.session_cache
```

### Example: Wildcard Table Filters

```ini
[mysqld]
replicate_wild_ignore_table = %.cache_%
replicate_wild_ignore_table = %.tmp_%
```

## Set Filters Dynamically (MySQL 8.0+)

You can add or remove filters at runtime without restarting:

```sql
STOP REPLICA SQL_THREAD;

CHANGE REPLICATION FILTER
  REPLICATE_DO_DB = (production, inventory),
  REPLICATE_IGNORE_TABLE = (production.audit_log);

START REPLICA SQL_THREAD;
```

For multi-source replication, apply filters per channel:

```sql
CHANGE REPLICATION FILTER
  REPLICATE_DO_DB = (production)
FOR CHANNEL 'primary1';
```

## View Current Filters

```sql
SELECT FILTER_NAME, FILTER_RULE
FROM performance_schema.replication_applier_filters;
```

Or for global filters:

```sql
SHOW REPLICA STATUS\G
```

Look for fields like `Replicate_Do_DB`, `Replicate_Ignore_DB`, `Replicate_Do_Table`.

## Important Caveats

### Statement-Based Replication and Database Context

With statement-based replication, `replicate_do_db` and `replicate_ignore_db` check the **current default database**, not the database in the query. This means:

```sql
USE other_db;
INSERT INTO production.orders VALUES (...);
```

The insert would be **skipped** even though it targets `production`, because the current database is `other_db`.

**Use row-based replication** to avoid this problem:

```ini
[mysqld]
binlog_format = ROW
```

### Cross-Database Queries

Filters based on database names do not work reliably for cross-database queries in statement mode. Use table-level filters (`replicate_do_table`, `replicate_ignore_table`) with row-based replication for precise control.

## Summary

MySQL replication filters allow selective replication of databases and tables. Configure them in `my.cnf` or dynamically using `CHANGE REPLICATION FILTER`. Always use row-based replication when applying database-level filters to avoid unexpected behavior with cross-database queries.
