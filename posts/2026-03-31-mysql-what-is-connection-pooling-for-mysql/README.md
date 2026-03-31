# What Is Connection Pooling for MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Connection Pooling, Performance, Scalability, Architecture

Description: Connection pooling for MySQL maintains a cache of reusable database connections to reduce the overhead of repeatedly establishing new connections for each request.

---

## Overview

Establishing a TCP connection, authenticating, and initializing a MySQL session takes time - typically 1-10ms per connection. In web applications handling hundreds of requests per second, creating a new connection for every request becomes a significant bottleneck. Connection pooling solves this by maintaining a pool of pre-established connections that can be reused across requests.

## How Connection Pooling Works

```text
Without pooling:
Request -> Open Connection -> Execute Query -> Close Connection -> Response
          (1-10ms overhead per request)

With pooling:
Request -> Borrow Connection from Pool -> Execute Query -> Return to Pool -> Response
           (microseconds - connection already open)
```

The pool manages:
- A minimum number of idle connections always ready
- A maximum cap to prevent overwhelming MySQL
- Connection validation and recycling for stale connections
- Queuing requests when the pool is exhausted

## MySQL's Connection Limit

Before pooling, check your MySQL server's connection capacity:

```sql
-- Current connection limits
SHOW VARIABLES LIKE 'max_connections';
SHOW VARIABLES LIKE 'max_user_connections';

-- Current active connections
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Threads_running';

-- Connection statistics
SHOW STATUS LIKE 'Connection_errors%';
SHOW STATUS LIKE 'Connections';
```

## Application-Level Pooling

Most MySQL drivers and frameworks include built-in pooling.

### Node.js with mysql2

```javascript
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
    host: 'localhost',
    user: 'appuser',
    password: 'secret',
    database: 'myapp',
    waitForConnections: true,
    connectionLimit: 20,       // Max connections in pool
    queueLimit: 0,             // Unlimited queue (0 = no limit)
    idleTimeout: 60000,        // Close idle connections after 60s
    enableKeepAlive: true,
    keepAliveInitialDelay: 0
});

// Use the pool - connections are automatically managed
async function getUser(userId) {
    const [rows] = await pool.execute(
        'SELECT id, username, email FROM users WHERE id = ?',
        [userId]
    );
    return rows[0];
}

// Pool handles acquiring and releasing connections
module.exports = pool;
```

### Python with SQLAlchemy

```python
from sqlalchemy import create_engine, text

engine = create_engine(
    "mysql+pymysql://appuser:secret@localhost/myapp",
    pool_size=10,           # Core pool size
    max_overflow=20,        # Extra connections beyond pool_size
    pool_timeout=30,        # Seconds to wait for a connection
    pool_recycle=3600,      # Recycle connections older than 1 hour
    pool_pre_ping=True      # Validate connection before use
)

def get_user(user_id: int):
    with engine.connect() as conn:
        result = conn.execute(
            text("SELECT id, username, email FROM users WHERE id = :id"),
            {"id": user_id}
        )
        return result.fetchone()
```

### Java with HikariCP

```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/myapp");
config.setUsername("appuser");
config.setPassword("secret");
config.setMaximumPoolSize(20);
config.setMinimumIdle(5);
config.setIdleTimeout(600000);       // 10 minutes
config.setConnectionTimeout(30000);  // 30 seconds
config.setMaxLifetime(1800000);      // 30 minutes
config.addDataSourceProperty("useSSL", "false");
config.addDataSourceProperty("cachePrepStmts", "true");
config.addDataSourceProperty("prepStmtCacheSize", "250");

HikariDataSource dataSource = new HikariDataSource(config);
```

## Proxy-Based Pooling with ProxySQL

For multi-application environments, a proxy pooler like ProxySQL sits between applications and MySQL:

```text
App 1 --\
App 2 ---+---> ProxySQL (connection pool) ---> MySQL Primary
App 3 --/              |
                       +---> MySQL Replica
```

```bash
# Connect to ProxySQL admin interface
mysql -h 127.0.0.1 -P 6032 -u admin -padmin

-- Check connection pool status
SELECT * FROM stats_mysql_connection_pool\G
```

## Monitoring Pool Health

```sql
-- Check for "Too many connections" errors
SHOW STATUS LIKE 'Connection_errors_max_connections';

-- Monitor threads over time
SELECT
    VARIABLE_NAME,
    VARIABLE_VALUE
FROM performance_schema.global_status
WHERE VARIABLE_NAME IN (
    'Threads_connected',
    'Threads_running',
    'Threads_cached',
    'Max_used_connections'
);
```

## Sizing the Pool

A common mistake is making pools too large. A good starting formula:

```text
pool_size = (core_count * 2) + effective_spindle_count
```

For a 4-core server with SSDs:
- pool_size = (4 * 2) + 1 = 9 connections per application instance

Too many connections cause context switching overhead on MySQL. Test with realistic load to find the sweet spot.

## Connection Validation

Always enable connection validation to detect stale connections:

```sql
-- MySQL keepalive: set wait_timeout appropriately
SHOW VARIABLES LIKE 'wait_timeout';
SHOW VARIABLES LIKE 'interactive_timeout';

-- Increase if your pool recycles before the server closes connections
SET GLOBAL wait_timeout = 28800;  -- 8 hours
```

## Summary

Connection pooling is essential for any MySQL application handling more than a few concurrent requests. It reduces connection overhead, improves throughput, and protects MySQL from being overwhelmed. Application-level pools (HikariCP, SQLAlchemy, mysql2) work well for single-application setups, while proxy-level pools like ProxySQL are better for multi-service architectures. Always validate connections before use and size your pool based on CPU cores rather than request volume.
