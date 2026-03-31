# How to Kill a Long-Running Query in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Query, Performance, Administration, Process

Description: Learn how to identify and kill long-running queries in MySQL using SHOW PROCESSLIST, KILL, and performance_schema to reclaim server resources.

---

Long-running queries consume database resources - CPU, memory, and locks - that prevent other queries from completing. Knowing how to safely identify and terminate these queries is an essential MySQL administration skill.

## Identifying Long-Running Queries

Use `SHOW PROCESSLIST` to list all active connections and their current queries:

```sql
SHOW FULL PROCESSLIST;
```

The output shows each connection with columns including `Id`, `User`, `Host`, `db`, `Command`, `Time` (in seconds), `State`, and `Info` (the query text). Look for rows with a high `Time` value.

For a filtered view showing only queries running longer than 30 seconds:

```sql
SELECT id, user, host, db, command, time, state, info
FROM information_schema.processlist
WHERE command != 'Sleep'
  AND time > 30
ORDER BY time DESC;
```

## Using performance_schema for More Detail

The `performance_schema.events_statements_current` table provides richer information:

```sql
SELECT t.processlist_id,
       t.processlist_user,
       t.processlist_db,
       t.processlist_time,
       t.processlist_state,
       s.sql_text
FROM performance_schema.threads t
JOIN performance_schema.events_statements_current s
  ON t.thread_id = s.thread_id
WHERE t.processlist_command != 'Sleep'
  AND t.processlist_time > 30
ORDER BY t.processlist_time DESC;
```

## Killing a Query

Once you have identified the process ID of the long-running query, use the `KILL` command:

```sql
KILL QUERY 12345;
```

`KILL QUERY` terminates the query but leaves the connection open. The client receives an error and can issue new queries.

To kill both the query and the connection:

```sql
KILL CONNECTION 12345;
-- or simply:
KILL 12345;
```

## Killing Multiple Long-Running Queries

To generate KILL statements for all queries running longer than 60 seconds:

```sql
SELECT CONCAT('KILL QUERY ', id, ';') AS kill_cmd
FROM information_schema.processlist
WHERE command != 'Sleep'
  AND time > 60;
```

Copy and execute the resulting statements. Alternatively, use a stored procedure:

```sql
DELIMITER //
CREATE PROCEDURE kill_long_queries(IN max_time INT)
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE proc_id BIGINT;
    DECLARE cur CURSOR FOR
        SELECT id FROM information_schema.processlist
        WHERE command != 'Sleep' AND time > max_time;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    OPEN cur;
    read_loop: LOOP
        FETCH cur INTO proc_id;
        IF done THEN LEAVE read_loop; END IF;
        KILL QUERY proc_id;
    END LOOP;
    CLOSE cur;
END //
DELIMITER ;

CALL kill_long_queries(60);
```

## Privileges Required

To kill your own queries, no special privileges are needed. To kill queries from other users, you need the `CONNECTION_ADMIN` privilege (MySQL 8.0) or the `SUPER` privilege:

```sql
GRANT CONNECTION_ADMIN ON *.* TO 'dba'@'localhost';
```

## Using mysqladmin from the Command Line

You can also kill a process from the shell without connecting to the MySQL client:

```bash
mysqladmin -u root -p kill 12345
```

## Summary

Killing long-running queries in MySQL involves finding the process ID with `SHOW FULL PROCESSLIST` or `information_schema.processlist`, then executing `KILL QUERY <id>` to stop the query while preserving the connection. Use `KILL CONNECTION <id>` when you want to fully disconnect the client. For recurring problems, investigate the root cause - missing indexes, lock contention, or missing query timeouts - rather than relying on manual intervention.
