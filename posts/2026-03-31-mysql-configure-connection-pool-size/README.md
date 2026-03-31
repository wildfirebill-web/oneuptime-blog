# How to Configure MySQL Connection Pool Size

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Connection Pool, Configuration, Performance, Tuning

Description: Learn how to calculate and configure the optimal MySQL connection pool size to maximize throughput while avoiding connection exhaustion and contention.

---

## Introduction

Configuring the wrong connection pool size is one of the most common MySQL performance mistakes. A pool that is too small causes threads to queue waiting for connections. A pool that is too large overwhelms MySQL with context switching and lock contention. This guide explains how to calculate the right size for your workload.

## The HikariCP Pool Sizing Formula

The HikariCP team popularized a counter-intuitive formula:

```text
pool_size = (core_count * 2) + effective_spindle_count
```

For a 4-core machine with an SSD (1 effective spindle):

```text
pool_size = (4 * 2) + 1 = 9
```

This is often much smaller than what developers expect. The reason is that CPU cores can only execute a fixed number of threads simultaneously - more connections beyond that point cause context switching overhead.

## Checking MySQL Server Limits

Before configuring pool size, check MySQL's maximum allowed connections:

```sql
SHOW VARIABLES LIKE 'max_connections';
SHOW STATUS LIKE 'Max_used_connections';
SHOW STATUS LIKE 'Threads_connected';
```

Your total pool connections across all application instances must not exceed `max_connections`.

## Example: Multi-Instance Configuration

If you have 3 application servers each with a pool:

```text
per_instance_pool = (total_mysql_max_connections - reserved_admin_connections) / num_app_servers
per_instance_pool = (500 - 10) / 3 = 163 connections per instance
```

## HikariCP Configuration

```java
config.setMaximumPoolSize(10);
config.setMinimumIdle(5);
config.setConnectionTimeout(30_000);
config.setIdleTimeout(600_000);
config.setMaxLifetime(1_800_000);
```

## SQLAlchemy (Python) Configuration

```python
from sqlalchemy import create_engine

engine = create_engine(
    "mysql+mysqlconnector://root:password@localhost/mydb",
    pool_size=9,
    max_overflow=5,
    pool_timeout=30,
    pool_recycle=1800,
)
```

`max_overflow` allows temporary additional connections beyond `pool_size` during traffic spikes.

## Node.js mysql2 Pool Configuration

```javascript
const pool = mysql.createPool({
  host: 'localhost',
  user: 'root',
  password: 'password',
  database: 'mydb',
  connectionLimit: 10,
  waitForConnections: true,
  queueLimit: 100,
});
```

## Go database/sql Configuration

```go
db.SetMaxOpenConns(10)
db.SetMaxIdleConns(5)
db.SetConnMaxLifetime(5 * time.Minute)
db.SetConnMaxIdleTime(2 * time.Minute)
```

## Setting MySQL's max_connections

Align MySQL's server limit with your pool sizes:

```sql
SET GLOBAL max_connections = 500;
```

Or in `my.cnf`:

```ini
[mysqld]
max_connections = 500
```

Each connection consumes roughly 1 MB of RAM on the MySQL server, so:

```text
max memory for connections = max_connections * ~1 MB
```

## Monitoring to Validate Pool Size

After deployment, verify your pool size is appropriate:

```sql
-- Connection count over time
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Threads_running';

-- Has the limit ever been hit?
SHOW STATUS LIKE 'Connection_errors_max_connections';
```

If `Connection_errors_max_connections` is non-zero, increase either MySQL's `max_connections` or the pool size.

## Summary

Start with the formula `(cores * 2) + spindles` as your initial pool size. Validate under load by monitoring `Threads_running` and `WaitCount` in your pool library's metrics. A healthy pool shows low wait counts and `Threads_running` significantly below `max_connections`. Avoid setting `max_connections` above what your MySQL server's RAM can handle comfortably.
