# How to Configure Connection Multiplexing with ProxySQL for MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, ProxySQL, Connection Pooling, Performance, Database

Description: Learn how to configure connection multiplexing in ProxySQL to reduce MySQL backend connections, improve throughput, and lower resource usage on your database server.

---

Connection multiplexing in ProxySQL allows many client connections to share a smaller pool of backend MySQL connections. This is one of ProxySQL's most powerful features for scaling MySQL applications under high concurrency.

## How Connection Multiplexing Works

ProxySQL maintains persistent backend connections to MySQL and reuses them across multiple client sessions. When a client sends a query, ProxySQL routes it to an available backend connection, then returns that connection to the pool. This means 10,000 client connections can share just 100 backend connections.

Multiplexing is enabled by default in ProxySQL but requires certain conditions: the client must not use session-level state that would prevent connection reuse, such as active transactions, temporary tables, or user-defined variables.

## Installing and Starting ProxySQL

```bash
# On Ubuntu/Debian
wget https://github.com/sysown/proxysql/releases/download/v2.5.5/proxysql_2.5.5-ubuntu22_amd64.deb
dpkg -i proxysql_2.5.5-ubuntu22_amd64.deb
systemctl start proxysql
```

Connect to the ProxySQL admin interface:

```bash
mysql -u admin -padmin -h 127.0.0.1 -P 6032
```

## Adding MySQL Backend Servers

```sql
-- Add MySQL backends
INSERT INTO mysql_servers (hostgroup_id, hostname, port, max_connections)
VALUES (10, '192.168.1.10', 3306, 200);

INSERT INTO mysql_servers (hostgroup_id, hostname, port, max_connections)
VALUES (10, '192.168.1.11', 3306, 200);

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

## Configuring the Connection Pool for Multiplexing

```sql
-- Set multiplexing-friendly pool variables
UPDATE mysql_variables SET variable_value = '50'
WHERE variable_name = 'mysql-free_connections_pct';

UPDATE mysql_variables SET variable_value = '1000'
WHERE variable_name = 'mysql-max_connections';

UPDATE mysql_variables SET variable_value = '3600000'
WHERE variable_name = 'mysql-connection_max_age_ms';

LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;
```

## Enabling and Tuning Multiplexing per Query Rule

ProxySQL evaluates whether a connection can be multiplexed based on session state. You can explicitly control this with query rules:

```sql
-- Allow multiplexing for read-only queries
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, multiplex)
VALUES (100, 1, '^SELECT', 20, 1);

-- Disable multiplexing for transactions
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, multiplex)
VALUES (200, 1, '^BEGIN', 10, 0);

LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

The `multiplex` field accepts: `0` (disable), `1` (enable), `2` (reset multiplexing after query).

## Monitoring Multiplexing Effectiveness

Check how well ProxySQL is reusing backend connections:

```sql
SELECT hostgroup, srv_host, ConnUsed, ConnFree, ConnOK, ConnERR, MaxConnUsed
FROM stats_mysql_connection_pool;
```

A high `ConnFree` and low `MaxConnUsed` relative to client connections confirms multiplexing is working. Also check:

```sql
SELECT * FROM stats_mysql_global
WHERE variable_name IN ('Client_Connections_connected', 'Backend_query_num_init');
```

## Conditions That Break Multiplexing

The following session-level operations disable multiplexing for that connection:

- `SET` statements that change session variables
- `LOCK TABLES`
- Multi-statement transactions without autocommit
- User-defined functions and stored procedures using `GET_LOCK()`

To verify which connections are being multiplexed:

```sql
SELECT * FROM stats_mysql_processlist WHERE MultiplexDisabled = 1;
```

## Summary

ProxySQL connection multiplexing lets thousands of application connections share a small pool of persistent MySQL backend connections, significantly reducing load on the database server. Configure `free_connections_pct` and `max_connections`, use query rules to control multiplexing per query type, and monitor `stats_mysql_connection_pool` to verify the pool is working efficiently.
