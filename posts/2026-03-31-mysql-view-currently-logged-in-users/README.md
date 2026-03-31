# How to View Currently Logged In Users in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, User, Monitoring, Database Administration, Performance Schema

Description: Learn how to see who is currently connected to MySQL using SHOW PROCESSLIST, information_schema, and performance_schema to monitor active sessions.

---

## Using SHOW PROCESSLIST

The simplest way to see currently connected users and their active queries:

```sql
SHOW PROCESSLIST;
```

Sample output:

```text
+----+-------+-----------+------+---------+------+----------+------------------+
| Id | User  | Host      | db   | Command | Time | State    | Info             |
+----+-------+-----------+------+---------+------+----------+------------------+
|  1 | root  | localhost | NULL | Query   |    0 | starting | SHOW PROCESSLIST |
|  7 | alice | 10.0.0.5  | mydb | Sleep   |  120 |          | NULL             |
| 12 | bob   | 10.0.0.8  | mydb | Query   |    2 | Sending  | SELECT ...       |
+----+-------+-----------+------+---------+------+----------+------------------+
```

Columns:
- `Id` - connection/thread ID
- `User` - authenticated username
- `Host` - client hostname or IP and port
- `db` - currently selected database
- `Command` - current command type (Query, Sleep, Connect, etc.)
- `Time` - seconds in current state
- `Info` - SQL query being executed (NULL if sleeping)

## SHOW FULL PROCESSLIST

The standard `SHOW PROCESSLIST` truncates the `Info` column to 100 characters. Use FULL for complete queries:

```sql
SHOW FULL PROCESSLIST;
```

## Querying information_schema.PROCESSLIST

For programmatic access or filtering:

```sql
SELECT ID, USER, HOST, DB, COMMAND, TIME, STATE, INFO
FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep'
ORDER BY TIME DESC;
```

Find connections from a specific user:

```sql
SELECT ID, HOST, DB, COMMAND, TIME
FROM information_schema.PROCESSLIST
WHERE USER = 'alice';
```

## Using performance_schema for Richer Details

The `performance_schema` provides more detail than PROCESSLIST:

```sql
SELECT t.PROCESSLIST_ID,
       t.PROCESSLIST_USER,
       t.PROCESSLIST_HOST,
       t.PROCESSLIST_DB,
       t.PROCESSLIST_COMMAND,
       t.PROCESSLIST_TIME,
       t.PROCESSLIST_STATE,
       t.PROCESSLIST_INFO
FROM performance_schema.threads t
WHERE t.PROCESSLIST_COMMAND IS NOT NULL
  AND t.TYPE = 'FOREGROUND'
ORDER BY t.PROCESSLIST_TIME DESC;
```

## Counting Active Connections per User

```sql
SELECT USER, COUNT(*) AS connections
FROM information_schema.PROCESSLIST
GROUP BY USER
ORDER BY connections DESC;
```

## Counting Total Active Connections

```sql
SHOW STATUS LIKE 'Threads_connected';
```

Or:

```sql
SELECT VARIABLE_VALUE AS active_connections
FROM performance_schema.global_status
WHERE VARIABLE_NAME = 'Threads_connected';
```

## Killing a Specific Connection

If you need to terminate a connection (for example, a long-running query):

```sql
-- Find the connection ID first
SHOW PROCESSLIST;

-- Kill the connection
KILL CONNECTION 12;

-- Kill only the current query (connection stays open)
KILL QUERY 12;
```

## Monitoring Long-Running Queries

```sql
SELECT ID, USER, HOST, DB, TIME, INFO
FROM information_schema.PROCESSLIST
WHERE COMMAND = 'Query'
  AND TIME > 30  -- running for more than 30 seconds
ORDER BY TIME DESC;
```

## Summary

`SHOW PROCESSLIST` and `information_schema.PROCESSLIST` are the primary tools for viewing currently connected MySQL users and their active queries. Use `SHOW FULL PROCESSLIST` for untruncated query text, query `performance_schema.threads` for richer session metadata, and `SHOW STATUS LIKE 'Threads_connected'` for a quick count. Use `KILL CONNECTION id` to terminate specific sessions that are blocking or consuming excessive resources.
