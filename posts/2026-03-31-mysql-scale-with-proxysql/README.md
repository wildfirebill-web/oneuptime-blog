# How to Scale MySQL with ProxySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, ProxySQL, Scaling, Load Balancing, Read Replica

Description: Learn how to scale MySQL using ProxySQL as a transparent proxy for connection pooling, read-write splitting, and query routing across multiple backends.

---

## Why ProxySQL Scales MySQL

ProxySQL is an open-source MySQL proxy that sits between your application and MySQL servers. It provides connection multiplexing, read-write splitting, query caching, and failover - all without application code changes. A single ProxySQL instance can multiplex thousands of application connections into a much smaller pool of backend MySQL connections, dramatically increasing effective capacity.

## Installing ProxySQL

On Ubuntu:

```bash
apt-get install -y proxysql2
systemctl enable proxysql
systemctl start proxysql
```

Connect to the ProxySQL admin interface (port 6032):

```bash
mysql -u admin -padmin -h 127.0.0.1 -P 6032
```

## Configuring Backend MySQL Servers

Add your MySQL primary (hostgroup 0) and replica (hostgroup 1):

```sql
INSERT INTO mysql_servers (hostgroup_id, hostname, port, max_connections, weight)
VALUES
  (0, '10.0.0.1', 3306, 200, 1),
  (1, '10.0.0.2', 3306, 200, 1),
  (1, '10.0.0.3', 3306, 200, 1);

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

## Configuring Read-Write Splitting

Create query rules to route SELECT statements to the read hostgroup (1) and everything else to the write hostgroup (0):

```sql
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES
  (1, 1, '^SELECT .* FOR UPDATE', 0, 1),
  (2, 1, '^SELECT', 1, 1);

LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

`SELECT ... FOR UPDATE` must go to the primary to avoid locking rows on a replica.

## Creating a ProxySQL MySQL User

```sql
INSERT INTO mysql_users (username, password, default_hostgroup, transaction_persistent)
VALUES ('app', 'secret', 0, 1);

LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;
```

`transaction_persistent = 1` ensures all statements within a transaction go to the same hostgroup, preventing reads from stale replicas mid-transaction.

## Configuring the Backend Monitoring User

ProxySQL monitors backend health using a dedicated user:

```sql
-- On MySQL primary:
CREATE USER 'proxysql_monitor'@'%' IDENTIFIED BY 'monitor_pass';
GRANT REPLICATION CLIENT ON *.* TO 'proxysql_monitor'@'%';

-- In ProxySQL admin:
SET mysql-monitor_username = 'proxysql_monitor';
SET mysql-monitor_password = 'monitor_pass';
LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;
```

## Testing the Connection

Applications connect to ProxySQL on port 6033:

```bash
mysql -u app -psecret -h 127.0.0.1 -P 6033 -e "SELECT @@hostname"
```

## Monitoring ProxySQL Statistics

Check connection pool utilization and query routing:

```sql
SELECT hostgroup, srv_host, ConnUsed, ConnFree, Queries
FROM stats.stats_mysql_connection_pool;

SELECT hostgroup, sum_time, count_star, digest_text
FROM stats.stats_mysql_query_digest
ORDER BY sum_time DESC
LIMIT 10;
```

## Summary

ProxySQL scales MySQL by transparently multiplexing application connections, routing reads to replicas and writes to the primary, and monitoring backend health for automatic failover. Install ProxySQL, configure backend server hostgroups and query rules, and connect applications to port 6033 instead of MySQL directly. Monitor via the stats schema to verify query routing and connection pool efficiency.
