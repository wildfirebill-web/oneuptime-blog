# How to Configure wait_timeout and interactive_timeout in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Configuration, Connection, Timeout, Performance

Description: Learn how to configure MySQL wait_timeout and interactive_timeout to manage idle connections, prevent connection leaks, and reduce resource waste.

---

## What Are These Timeouts

MySQL uses two timeout variables to close idle connections:

- **`wait_timeout`** - seconds the server waits on a non-interactive connection before closing it (used by applications and connection pools)
- **`interactive_timeout`** - seconds the server waits on an interactive connection (such as a `mysql` CLI session) before closing it

When a connection is idle for longer than the applicable timeout, MySQL closes it automatically. Any subsequent query on that connection returns `ERROR 2006: MySQL server has gone away`.

## Check Current Values

```sql
SHOW VARIABLES LIKE '%timeout%';
```

```text
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| interactive_timeout        | 28800 |
| wait_timeout               | 28800 |
+----------------------------+-------+
```

The default is 28800 seconds (8 hours).

## When to Change These Values

**Lower the timeout when:**
- You have `Too many connections` errors due to idle connections piling up
- Application connection pools are not properly closing connections
- You want to free resources from long-running client sessions

**Raise the timeout when:**
- You have batch jobs that run queries spaced more than 8 hours apart
- You are using persistent connections without a keep-alive mechanism

## Set Timeouts Globally

```sql
-- Set for all new connections (existing connections unaffected)
SET GLOBAL wait_timeout = 3600;
SET GLOBAL interactive_timeout = 3600;
```

## Set Timeout for Current Session

```sql
SET SESSION wait_timeout = 300;
```

## Persist in my.cnf

```ini
[mysqld]
wait_timeout = 3600
interactive_timeout = 3600
```

## Recommended Values

| Scenario | wait_timeout | interactive_timeout |
|----------|-------------|---------------------|
| Web app with connection pool | 600-1800 | 3600 |
| Long-running batch jobs | 28800 | 28800 |
| Cloud/serverless (short-lived) | 60-300 | 300 |

## Handle Timeout in Application Code

Configure your connection pool to validate connections before use:

```python
# SQLAlchemy example - reconnect if connection dropped
from sqlalchemy import create_engine

engine = create_engine(
    'mysql+pymysql://user:pass@localhost/mydb',
    pool_pre_ping=True,          # test connection before using it
    pool_recycle=1800,           # recycle connections after 30 minutes
    pool_size=10,
    max_overflow=20
)
```

For Node.js with `mysql2`:

```javascript
const pool = mysql.createPool({
  host: 'localhost',
  user: 'app',
  password: 'secret',
  database: 'mydb',
  waitForConnections: true,
  connectionLimit: 10,
  enableKeepAlive: true,        // send keepalive packets
  keepAliveInitialDelay: 10000  // start keepalive after 10 seconds
});
```

## Find Idle Connections

```sql
SELECT ID, USER, HOST, DB, COMMAND, TIME, STATE
FROM information_schema.processlist
WHERE COMMAND = 'Sleep'
ORDER BY TIME DESC;
```

```text
+----+------+----------------+-------+---------+------+-------+
| ID | USER | HOST           | DB    | COMMAND | TIME | STATE |
+----+------+----------------+-------+---------+------+-------+
| 45 | app  | web1:54321     | mydb  | Sleep   | 7200 |       |
+----+------+----------------+-------+---------+------+-------+
```

A high TIME value on a Sleep connection indicates the connection is idle and will be killed at `wait_timeout`.

## Summary

Configure `wait_timeout` (for application connections) and `interactive_timeout` (for CLI sessions) in MySQL to control how long idle connections persist. Lower values (600-1800 seconds) are appropriate for web applications using connection pools. Configure pool libraries with `pool_pre_ping` or connection recycling to handle connections closed by MySQL after the timeout.
