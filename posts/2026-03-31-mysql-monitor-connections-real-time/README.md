# How to Monitor MySQL Connections in Real Time

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Connection, Monitoring, Performance, Administration

Description: Learn how to monitor MySQL connections in real time using SHOW PROCESSLIST, performance_schema, and system tools to identify connection leaks, idle sessions, and bottlenecks.

---

Monitoring MySQL connections in real time helps you detect connection leaks, identify long-running queries, and prevent hitting `max_connections`. MySQL provides several built-in tools for this without requiring any external software.

## Checking Current Connection Count

The fastest way to see active connections:

```sql
SHOW STATUS LIKE 'Threads_connected';
```

To also see peak connections since last restart:

```sql
SHOW STATUS LIKE 'Max_used_connections';
SHOW VARIABLES LIKE 'max_connections';
```

This tells you current load vs capacity. If `Threads_connected` is close to `max_connections`, you are near the limit.

## Viewing Active Connections with SHOW PROCESSLIST

```sql
SHOW FULL PROCESSLIST;
```

This shows all active connections with their state, query, and duration:

```
+------+------+-----------+------+---------+------+-------+------------------+
| Id   | User | Host      | db   | Command | Time | State | Info             |
+------+------+-----------+------+---------+------+-------+------------------+
| 1    | app  | 10.0.0.5  | mydb | Query   | 0    |       | SELECT * FROM... |
| 2    | app  | 10.0.0.6  | mydb | Sleep   | 3600 |       | NULL             |
+------+------+-----------+------+---------+------+-------+------------------+
```

Connections in `Sleep` state for a long time are idle connections that may indicate a connection leak in the application.

## Querying the Performance Schema

The `performance_schema.threads` table provides more detail:

```sql
SELECT thread_id, name, type, processlist_user, processlist_host,
       processlist_db, processlist_command, processlist_time, processlist_state
FROM performance_schema.threads
WHERE type = 'FOREGROUND'
ORDER BY processlist_time DESC
LIMIT 20;
```

For connection counts grouped by host:

```sql
SELECT processlist_host AS host, COUNT(*) AS connections
FROM performance_schema.threads
WHERE type = 'FOREGROUND'
GROUP BY processlist_host
ORDER BY connections DESC;
```

## Identifying Long-Running Queries

```sql
SELECT id, user, host, db, command, time, state, LEFT(info, 100) AS query
FROM information_schema.processlist
WHERE command != 'Sleep'
  AND time > 30
ORDER BY time DESC;
```

This finds queries running longer than 30 seconds, which may be causing blocking or consuming excessive resources.

## Watching Connection Changes Continuously

You can poll the connection status repeatedly from the shell:

```bash
watch -n 2 "mysql -u root -p'password' -e 'SHOW STATUS LIKE \"%connections%\";'"
```

Or use this one-liner to output a rolling count:

```bash
while true; do
  mysql -u root -p'password' -se 'SELECT NOW(), COUNT(*) FROM information_schema.processlist;'
  sleep 2
done
```

## Killing Idle or Problem Connections

To kill all connections that have been idle for more than one hour:

```sql
-- Generate kill statements for idle connections
SELECT CONCAT('KILL ', id, ';')
FROM information_schema.processlist
WHERE command = 'Sleep' AND time > 3600;
```

Run the output of that query, or use this procedure to kill them automatically:

```sql
DELIMITER //
CREATE PROCEDURE kill_idle_connections(IN max_idle_seconds INT)
BEGIN
  DECLARE done INT DEFAULT FALSE;
  DECLARE conn_id BIGINT;
  DECLARE cur CURSOR FOR
    SELECT id FROM information_schema.processlist
    WHERE command = 'Sleep' AND time > max_idle_seconds;
  DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
  OPEN cur;
  read_loop: LOOP
    FETCH cur INTO conn_id;
    IF done THEN LEAVE read_loop; END IF;
    KILL conn_id;
  END LOOP;
  CLOSE cur;
END //
DELIMITER ;

CALL kill_idle_connections(3600);
```

## Monitoring with mysqladmin

For quick command-line monitoring:

```bash
# Show status every 2 seconds
mysqladmin -u root -p status --sleep=2

# Show processlist
mysqladmin -u root -p processlist
```

## Summary

Monitor MySQL connections in real time using `SHOW FULL PROCESSLIST` for a quick snapshot, `performance_schema.threads` for detailed per-connection data, and `information_schema.processlist` for filtered queries. Track `Threads_connected` against `max_connections` to watch capacity, and periodically kill long-idle connections to prevent connection pool exhaustion. Use `watch` or a polling loop for continuous monitoring from the shell.
