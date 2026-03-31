# How to Scale MySQL with Connection Pooling

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Connection Pooling, Scaling, Performance, ProxySQL

Description: Learn how to scale MySQL connection handling using application-level and proxy-level connection pooling to reduce overhead and support higher concurrency.

---

## The Cost of MySQL Connections

Every MySQL connection spawns a thread and consumes memory (roughly 4-8 MB per connection for stack and buffers). With hundreds of application processes each opening their own connections, MySQL can exhaust memory or hit `max_connections` limits long before it saturates CPU or I/O.

Connection pooling solves this by maintaining a pool of pre-established connections that are reused across application requests, dramatically reducing connection overhead.

## Application-Level Pooling

### Node.js with mysql2

```javascript
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
  host: 'db.example.com',
  user: 'app',
  password: 'secret',
  database: 'mydb',
  waitForConnections: true,
  connectionLimit: 20,
  queueLimit: 100,
  idleTimeout: 60000,
  enableKeepAlive: true,
  keepAliveInitialDelay: 30000
});

async function getUser(userId) {
  const [rows] = await pool.execute(
    'SELECT * FROM users WHERE id = ?', [userId]
  );
  return rows[0];
}
```

### Python with SQLAlchemy

```python
from sqlalchemy import create_engine

engine = create_engine(
    "mysql+mysqlconnector://app:secret@db.example.com/mydb",
    pool_size=20,
    max_overflow=10,
    pool_timeout=30,
    pool_recycle=1800,
    pool_pre_ping=True
)
```

`pool_pre_ping=True` runs a lightweight `SELECT 1` before using a connection to detect and replace stale connections.

## Proxy-Level Pooling with ProxySQL

ProxySQL is a dedicated MySQL proxy that provides connection pooling, query routing, and failover. It sits between your application and MySQL:

```text
Application -> ProxySQL (port 6033) -> MySQL Primary/Replicas
```

Install and configure ProxySQL:

```bash
apt-get install proxysql2
systemctl start proxysql
```

Connect to the ProxySQL admin interface and configure backend servers:

```sql
-- Connect to ProxySQL admin (port 6032)
INSERT INTO mysql_servers (hostgroup_id, hostname, port, max_connections)
VALUES (0, '10.0.0.1', 3306, 200);

INSERT INTO mysql_users (username, password, default_hostgroup)
VALUES ('app', 'secret', 0);

LOAD MYSQL SERVERS TO RUNTIME;
LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
SAVE MYSQL USERS TO DISK;
```

ProxySQL multiplexes many application connections into a smaller pool of MySQL connections, reducing the actual connection count on MySQL.

## Setting MySQL max_connections Correctly

With pooling, MySQL `max_connections` can be set lower because the proxy or application pool limits actual connections:

```ini
[mysqld]
max_connections = 500
```

Monitor current connections:

```sql
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';
```

If `Max_used_connections` is consistently close to `max_connections`, increase the limit or reduce the pool size.

## Detecting Connection Pool Exhaustion

When all pool connections are busy, new requests queue or fail. Monitor pool wait time and pool exhaustion:

```sql
-- Check for connection wait errors
SHOW GLOBAL STATUS LIKE 'Connection_errors_max_connections';
SHOW GLOBAL STATUS LIKE 'Aborted_connects';
```

In ProxySQL, query the stats table:

```sql
SELECT hostgroup, srv_host, ConnUsed, ConnFree, ConnOK
FROM stats.stats_mysql_connection_pool;
```

## Summary

Connection pooling scales MySQL by reusing established connections across application threads, reducing memory consumption and connection overhead. Use application-level pools (mysql2 in Node.js, SQLAlchemy in Python) for simple deployments, or add ProxySQL as a proxy-level pool for multi-application environments. Monitor `Threads_connected` and `Max_used_connections` to size your pool correctly.
