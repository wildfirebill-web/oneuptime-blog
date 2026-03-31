# How to Handle MySQL Connection Management Best Practices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Connection, Pool, Performance, Best Practice

Description: Learn MySQL connection management best practices including connection pooling, pool sizing, timeout configuration, and graceful handling of lost connections.

---

MySQL connections are expensive to establish - each one spawns a thread on the server. Poor connection management leads to "Too many connections" errors, connection leak exhaustion, and unnecessarily high server resource usage. A disciplined approach to pooling and lifecycle management prevents these problems.

## Always Use a Connection Pool

Opening a new TCP connection for every query is impractical at any meaningful load. Use a connection pool that reuses existing connections:

```python
# Python with SQLAlchemy
from sqlalchemy import create_engine

engine = create_engine(
    "mysql+pymysql://app_user:secret@db-host/myapp",
    pool_size=10,          # persistent connections kept open
    max_overflow=5,        # temporary extra connections above pool_size
    pool_timeout=30,       # seconds to wait for a connection from the pool
    pool_recycle=1800,     # recycle connections older than 30 minutes
    pool_pre_ping=True     # test connectivity before using a connection
)
```

`pool_pre_ping=True` is critical: it issues a lightweight `SELECT 1` before handing a connection to your code, preventing failures from stale connections that were silently closed by a firewall or the server.

## Size the Pool Correctly

A common mistake is setting the pool too large. MySQL defaults to 151 max connections. Each connection uses memory (~1-8 MB). The formula for pool size per application instance is:

```text
pool_size = (server_max_connections - reserved_admin_connections) / number_of_app_instances
```

For a server with 200 max connections and 3 app instances:

```text
pool_size = (200 - 5) / 3 = 65 per instance
```

Check current usage before sizing:

```sql
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';
SHOW VARIABLES LIKE 'max_connections';
```

## Configure Server-Side Timeouts

MySQL closes idle connections after `wait_timeout` seconds (default 28800 - 8 hours). Set it lower in `my.cnf` to reclaim resources faster:

```ini
[mysqld]
wait_timeout         = 300
interactive_timeout  = 300
```

On the client side, set `pool_recycle` below `wait_timeout` to ensure the pool retires connections before the server kills them silently.

## Handle Connection Errors with Retry

Transient network interruptions can cause connection failures. Implement exponential backoff retry:

```python
import time
from sqlalchemy.exc import OperationalError

def execute_with_retry(engine, query, max_attempts=3):
    for attempt in range(1, max_attempts + 1):
        try:
            with engine.connect() as conn:
                return conn.execute(query).fetchall()
        except OperationalError as e:
            if attempt == max_attempts:
                raise
            sleep_time = 2 ** attempt
            time.sleep(sleep_time)
```

## Use a Separate Read Pool for Read Replicas

Separate write and read connections to distribute load:

```python
write_engine = create_engine("mysql+pymysql://user:pass@primary-host/myapp", pool_size=5)
read_engine  = create_engine("mysql+pymysql://user:pass@replica-host/myapp", pool_size=15)
```

Route SELECT queries to the read pool and write operations to the write pool explicitly in your data access layer.

## Monitor Active Connections

```sql
SELECT user, host, db, command, time, state
FROM information_schema.PROCESSLIST
WHERE command != 'Sleep'
ORDER BY time DESC;
```

## Summary

Effective MySQL connection management means using a connection pool sized to the server's capacity, setting pool recycle intervals below server-side timeouts, enabling pre-ping checks, and implementing retry logic for transient errors. Splitting read and write pools across replicas further reduces contention on the primary and keeps both pools right-sized for their workloads.
