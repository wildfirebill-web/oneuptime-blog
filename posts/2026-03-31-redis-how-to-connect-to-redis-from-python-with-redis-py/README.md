# How to Connect to Redis from Python with redis-py

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Python, redis-py, Connection Pooling, Getting Started

Description: Learn how to install redis-py, connect to Redis from Python, use connection pools, and perform basic operations with the official Python client.

---

## Installing redis-py

Install the official Redis Python client:

```bash
pip install redis
```

For additional features like hiredis parser (faster):

```bash
pip install redis[hiredis]
```

## Basic Connection

```python
import redis

# Connect to local Redis
r = redis.Redis(host='localhost', port=6379, db=0)

# Test the connection
pong = r.ping()
print(pong)  # True

# With password
r = redis.Redis(host='localhost', port=6379, password='yourpassword', db=0)

# Decode responses to strings (instead of bytes)
r = redis.Redis(host='localhost', port=6379, decode_responses=True)
```

## Connecting with a URL

```python
import redis

# Simple URL format
r = redis.from_url('redis://localhost:6379/0')

# With password
r = redis.from_url('redis://:yourpassword@localhost:6379/0')

# TLS connection
r = redis.from_url('rediss://localhost:6380/0')
```

## Using Connection Pools

Connection pools are recommended for production to avoid creating a new TCP connection for every command:

```python
import redis

pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    db=0,
    max_connections=20,
    decode_responses=True
)

r = redis.Redis(connection_pool=pool)

# Multiple Redis instances sharing the same pool
r1 = redis.Redis(connection_pool=pool)
r2 = redis.Redis(connection_pool=pool)
```

## Basic CRUD Operations

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# String operations
r.set('username', 'alice')
value = r.get('username')
print(value)  # alice

# Set with expiry (TTL in seconds)
r.setex('session:abc', 3600, 'user_data')

# Increment a counter
r.set('visits', 0)
r.incr('visits')
r.incrby('visits', 5)
count = r.get('visits')
print(count)  # 6

# Check key existence
exists = r.exists('username')
print(exists)  # 1

# Delete a key
r.delete('username')

# Set expiry on an existing key
r.expire('session:abc', 7200)

# Get TTL
ttl = r.ttl('session:abc')
print(ttl)  # seconds remaining
```

## Hash Operations

```python
# Store a user object as a hash
r.hset('user:1001', mapping={
    'name': 'Alice',
    'email': 'alice@example.com',
    'age': '30'
})

# Get a single field
name = r.hget('user:1001', 'name')
print(name)  # Alice

# Get all fields
user = r.hgetall('user:1001')
print(user)  # {'name': 'Alice', 'email': 'alice@example.com', 'age': '30'}

# Update a field
r.hset('user:1001', 'age', '31')

# Delete a field
r.hdel('user:1001', 'age')
```

## List Operations

```python
# Push items to a list
r.rpush('tasks', 'task1', 'task2', 'task3')

# Pop from the front (FIFO queue)
task = r.lpop('tasks')
print(task)  # task1

# Get all items
items = r.lrange('tasks', 0, -1)
print(items)  # ['task2', 'task3']

# Blocking pop (waits for item)
task = r.blpop('tasks', timeout=5)
```

## TLS Connection

```python
import redis
import ssl

r = redis.Redis(
    host='redis.example.com',
    port=6380,
    ssl=True,
    ssl_certfile='/path/to/client.crt',
    ssl_keyfile='/path/to/client.key',
    ssl_ca_certs='/path/to/ca.crt',
    decode_responses=True
)
```

## Handling Connection Errors

```python
import redis
from redis.exceptions import ConnectionError, TimeoutError, RedisError

r = redis.Redis(
    host='localhost',
    port=6379,
    socket_connect_timeout=5,
    socket_timeout=5,
    retry_on_timeout=True,
    decode_responses=True
)

try:
    r.ping()
    print("Connected successfully")
except ConnectionError as e:
    print(f"Could not connect to Redis: {e}")
except TimeoutError as e:
    print(f"Connection timed out: {e}")
except RedisError as e:
    print(f"Redis error: {e}")
```

## Summary

redis-py is the standard Python client for Redis, offering simple connection setup, URL-based configuration, and connection pooling for production use. By using `decode_responses=True` and connection pools with `max_connections`, you get string-based responses and efficient connection management. Always wrap Redis calls in error handling to gracefully manage connectivity issues in production applications.
