# How to Use SELECT in Redis to Switch Between Databases

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Database, Command, Namespacing, Configuration

Description: Learn how to use SELECT in Redis to switch between logical databases (0-15), understand their limitations in cluster mode, and when to use namespacing instead.

---

## What Is SELECT in Redis

Redis supports multiple logical databases numbered 0 through 15 (configurable via `databases` setting in redis.conf). `SELECT` switches the current connection to a specific database. Keys in different databases are completely isolated from each other.

```text
SELECT index
```

- `index` - database number (0 to `databases - 1`)

Returns `OK` on success.

## Basic Usage

```bash
# Default is database 0
SELECT 0

# Switch to database 1
SELECT 1
SET user:1 "Alice"

# Switch to database 2
SELECT 2
SET user:1 "Bob"   # Different from db 1's user:1

# Go back to db 1
SELECT 1
GET user:1
# "Alice"

SELECT 2
GET user:1
# "Bob"
```

## Configuring Number of Databases

In `redis.conf`:

```text
databases 16
```

This allows databases 0 through 15. You cannot select an index outside this range:

```bash
SELECT 16
# ERR DB index is out of range
```

## Practical Example in Python

```python
import redis

# Connect to specific database at connection time
db0 = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)
db1 = redis.Redis(host='localhost', port=6379, db=1, decode_responses=True)
db2 = redis.Redis(host='localhost', port=6379, db=2, decode_responses=True)

# Write to different databases
db0.set('app:config', 'production')
db1.set('session:token123', 'user:42')
db2.set('cache:homepage', '<html>...</html>')

# Keys are isolated
print(db0.get('session:token123'))  # None - not in db0
print(db1.get('session:token123'))  # "user:42"

# Switch database on existing connection
client = redis.Redis(host='localhost', port=6379, decode_responses=True)
client.select(1)
print(client.get('session:token123'))  # "user:42"
```

## Practical Example in Node.js

```javascript
const { createClient } = require('redis');

// Connect to specific database
const db0 = createClient({ database: 0 });
const db1 = createClient({ database: 1 });
const db3 = createClient({ database: 3 });

await db0.connect();
await db1.connect();
await db3.connect();

// Each database is isolated
await db0.set('key', 'from-db0');
await db1.set('key', 'from-db1');

console.log(await db0.get('key'));  // "from-db0"
console.log(await db1.get('key'));  // "from-db1"
console.log(await db3.get('key'));  // null - not set in db3
```

## Common Use Cases for Multiple Databases

```text
Database 0: Application data (default)
Database 1: Session storage
Database 2: Cache layer
Database 3: Rate limiting counters
Database 4: Background job queues
Database 15: Testing/development
```

## SELECT Does Not Work in Cluster Mode

Redis cluster mode only supports database 0. Using `SELECT` with any other index in cluster mode returns an error:

```bash
SELECT 1
# ERR SELECT is not allowed in cluster mode
```

In cluster mode, use key prefixes/namespaces for logical separation:

```bash
# Instead of SELECT 1
SET session:token123 user:42

# Instead of SELECT 2
SET cache:homepage "<html>..."
```

## FLUSHDB vs FLUSHALL with Multiple Databases

```bash
# Delete all keys in the current database only
SELECT 3
FLUSHDB
# Clears db 3, leaves other databases intact

# Delete all keys in ALL databases
FLUSHALL
# Clears everything!
```

## Moving Keys Between Databases

Use `MOVE` to relocate a key from the current database to another:

```bash
SELECT 0
SET mykey "hello"
MOVE mykey 2  # Move to database 2

GET mykey     # nil - gone from db 0
SELECT 2
GET mykey     # "hello" - now in db 2
```

## Checking Database Key Counts

```bash
# Count keys in current database
DBSIZE

SELECT 0
DBSIZE  # Keys in db 0

SELECT 1
DBSIZE  # Keys in db 1

# Info about all databases
INFO keyspace
# db0:keys=100,expires=10,avg_ttl=3600000
# db1:keys=50,expires=50,avg_ttl=1800000
```

## When to Avoid Multiple Databases

- **Cluster mode** - not supported
- **Replication** - replicas receive all databases; you can't replicate just one
- **Large teams** - hard to track which app uses which database
- **Microservices** - use separate Redis instances instead

Consider using key prefixes or separate Redis instances for better isolation:

```python
import redis

# Prefix-based namespacing (works in cluster mode too)
class NamespacedRedis:
    def __init__(self, client, namespace):
        self.client = client
        self.ns = namespace

    def set(self, key, value, **kwargs):
        return self.client.set(f"{self.ns}:{key}", value, **kwargs)

    def get(self, key):
        return self.client.get(f"{self.ns}:{key}")

client = redis.Redis(host='localhost', port=6379, decode_responses=True)
sessions = NamespacedRedis(client, 'session')
cache = NamespacedRedis(client, 'cache')

sessions.set('token123', 'user:42')
cache.set('homepage', '<html>...')
```

## Summary

`SELECT` switches a Redis connection to a specific logical database (0-15), providing key isolation without separate server instances. It is useful on standalone Redis for separating concerns like sessions, caches, and queues. However, it does not work in cluster mode, so use key namespacing or separate Redis instances when running in cluster deployments.
