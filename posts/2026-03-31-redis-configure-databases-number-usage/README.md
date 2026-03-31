# How to Configure Redis Databases (Number and Usage)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Database, Configuration

Description: Learn how to configure the number of Redis databases, switch between them, and understand when to use multiple databases vs separate instances.

---

Redis supports multiple logical databases within a single instance. By default Redis creates 16 databases, numbered 0 through 15. This setting is controlled by the `databases` directive in `redis.conf`.

## Setting the Number of Databases

Open your `redis.conf` file and find or add the `databases` directive:

```text
# redis.conf
databases 16
```

To reduce memory overhead on constrained systems or increase isolation, you can change this value:

```bash
# Reduce to 4 databases
sed -i 's/^databases .*/databases 4/' /etc/redis/redis.conf

# Restart Redis to apply
sudo systemctl restart redis
```

You can also set this at startup:

```bash
redis-server --databases 4
```

## Switching Between Databases

Use the `SELECT` command to switch the active database for your connection:

```bash
redis-cli
127.0.0.1:6379> SELECT 0
OK
127.0.0.1:6379> SET app:session "user123"
OK
127.0.0.1:6379> SELECT 1
OK
127.0.0.1:6379[1]> SET cache:products "laptop,phone"
OK
```

In Python with `redis-py`, specify the database index at connection time:

```python
import redis

# Connect to database 0 (default)
r0 = redis.Redis(host='localhost', port=6379, db=0)
r0.set('session:user1', 'active')

# Connect to database 1
r1 = redis.Redis(host='localhost', port=6379, db=1)
r1.set('cache:home', '{"items":[]}')
```

## Checking Database Usage

Use `INFO keyspace` to see which databases have keys:

```bash
redis-cli INFO keyspace
```

```text
# Keyspace
db0:keys=1024,expires=512,avg_ttl=3600000
db1:keys=256,expires=0,avg_ttl=0
```

To count keys in the current database:

```bash
redis-cli -n 0 DBSIZE
redis-cli -n 1 DBSIZE
```

## Limitations of Multiple Databases

Redis databases do not provide true isolation. All databases share:

- The same memory pool
- The same CPU thread
- The same persistence files (RDB/AOF)
- The same replication stream

When using Redis Cluster, only database 0 is supported. Attempting to SELECT another database returns an error:

```text
ERR SELECT is not allowed in cluster mode
```

## When to Use Multiple Databases

Multiple databases work well for:

- Separating application concerns (sessions vs cache vs queue) on a single small instance
- Development environments where you want to isolate test data
- Flush operations where you call `FLUSHDB` on one database without affecting others

For production multi-tenant systems or workloads with different memory policies, deploy separate Redis instances instead. This gives you independent persistence, memory limits, and replication.

## Flushing a Single Database

```bash
# Flush only database 1
redis-cli -n 1 FLUSHDB

# Flush all databases (dangerous in production)
redis-cli FLUSHALL
```

Use `FLUSHDB ASYNC` to avoid blocking Redis during large flushes:

```bash
redis-cli -n 1 FLUSHDB ASYNC
```

## Summary

Redis supports up to 16 logical databases by default, configurable via the `databases` directive. Databases share memory and I/O resources, so they are best used for lightweight logical separation within a single small instance. For production workloads requiring independent resource limits or cluster deployments, prefer separate Redis instances over multiple databases.
