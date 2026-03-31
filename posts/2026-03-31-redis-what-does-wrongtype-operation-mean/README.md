# What Does 'WRONGTYPE Operation' Mean in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Error, Debugging, Data Types

Description: Understand the Redis WRONGTYPE error, why it occurs when mixing data types on the same key, and how to fix it with practical examples.

---

The `WRONGTYPE Operation against a key holding the wrong kind of value` error is one of the most common mistakes when working with Redis. It occurs when you attempt to use a command designed for one data type on a key that holds a different type.

## What Causes the Error

Every Redis key holds exactly one data type: string, list, set, sorted set, hash, stream, or bitmap. If you store a string in a key and then try to use a list command on it, Redis returns the WRONGTYPE error.

```bash
127.0.0.1:6379> SET user:100 "alice"
OK
127.0.0.1:6379> LPUSH user:100 "item"
(error) WRONGTYPE Operation against a key holding the wrong kind of value
```

Here, `user:100` is a string. `LPUSH` is a list command, so Redis rejects it.

## Common Scenarios

**Mixing hash and string commands:**

```bash
127.0.0.1:6379> SET config:timeout 30
OK
127.0.0.1:6379> HSET config:timeout host "localhost"
(error) WRONGTYPE Operation against a key holding the wrong kind of value
```

**Using set commands on a sorted set:**

```bash
127.0.0.1:6379> ZADD leaderboard 100 "player1"
(integer) 1
127.0.0.1:6379> SMEMBERS leaderboard
(error) WRONGTYPE Operation against a key holding the wrong kind of value
```

## How to Diagnose the Issue

Use `TYPE` to inspect the current data type of a key:

```bash
127.0.0.1:6379> TYPE user:100
string
127.0.0.1:6379> TYPE leaderboard
zset
```

Use `DEBUG OBJECT` for more detail:

```bash
127.0.0.1:6379> DEBUG OBJECT user:100
Value at:0x7f... refcount:1 encoding:embstr serializedlength:5 lru:... type:string
```

## How to Fix It

**Option 1: Delete the key and recreate it with the correct type.**

```bash
127.0.0.1:6379> DEL user:100
(integer) 1
127.0.0.1:6379> LPUSH user:100 "item"
(integer) 1
```

**Option 2: Use the correct command for the existing type.**

If the key is a string and you want a list, rename or repurpose the key:

```bash
127.0.0.1:6379> RENAME user:100 user:100:legacy
OK
127.0.0.1:6379> LPUSH user:100:items "item"
(integer) 1
```

## Preventing the Error in Code

Check the type before performing operations, or use key naming conventions to encode the type.

```python
import redis

r = redis.Redis(decode_responses=True)

def safe_lpush(key, value):
    key_type = r.type(key)
    if key_type not in ('list', 'none'):
        raise ValueError(f"Key {key} is of type {key_type}, expected list")
    r.lpush(key, value)
```

Use consistent naming conventions like `user:100:items` (list), `user:100:meta` (hash), `user:100:score` (sorted set) to make the intended type explicit.

## Summary

The WRONGTYPE error in Redis means you are using a command that does not match the data type currently stored in that key. Use `TYPE` to inspect the key, delete and recreate it with the correct type, or restructure your key naming convention to prevent type collisions.
