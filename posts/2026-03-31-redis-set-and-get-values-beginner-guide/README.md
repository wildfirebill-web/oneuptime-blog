# How to Set and Get Values in Redis (Beginner Guide)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Beginner, SET, GET, String

Description: A beginner guide to setting and getting values in Redis - covering SET, GET, MSET, MGET, expiration options, and conditional writes.

---

The two most fundamental Redis commands are SET and GET. This guide covers everything a beginner needs to know about storing and retrieving values in Redis.

## The SET Command

```bash
# Basic syntax
SET key value

# Examples
SET name "Alice"
SET age "30"
SET message "Hello, Redis!"
```

After a SET, the key holds the value until you delete it, overwrite it, or it expires.

## The GET Command

```bash
# Basic syntax
GET key

# Examples
GET name
# "Alice"

GET age
# "30"

# Returns nil if key doesn't exist
GET nonexistent_key
# (nil)
```

## SET with Expiration (TTL)

You can make a key expire automatically after a certain time:

```bash
# Expire in seconds
SET session_token "abc123" EX 3600       # expires in 1 hour

# Expire in milliseconds
SET temp_code "987654" PX 30000          # expires in 30 seconds

# Expire at a specific Unix timestamp
SET promo_price "9.99" EXAT 1800000000

# Check remaining time
TTL session_token        # returns seconds
PTTL temp_code           # returns milliseconds
```

## Conditional SET (NX and XX)

```bash
# NX = only set if key does NOT exist
SET username "alice" NX
# OK  (if username didn't exist)

SET username "bob" NX
# (nil) - no-op, username already exists

# XX = only set if key DOES exist
SET username "carol" XX
# OK (if username already existed)

SET newkey "value" XX
# (nil) - no-op, newkey doesn't exist
```

## GET the Old Value While Setting a New One

```bash
# In Redis 6.2+, use GET option with SET
SET counter 10
SET counter 20 GET
# Returns "10" (the old value) and sets to 20
```

## Setting and Getting Multiple Keys

```bash
# MSET: set multiple keys in one command
MSET name "Alice" city "London" country "UK"
# OK

# MGET: get multiple keys in one command
MGET name city country
# 1) "Alice"
# 2) "London"
# 3) "UK"

# MSETNX: set multiple keys only if NONE of them exist
MSETNX key1 "a" key2 "b"
# (integer) 1  -> success (all keys were absent)
```

## GETSET and GETDEL

```bash
# GETSET (deprecated in Redis 6.2, use SET ... GET instead)
GETSET name "Bob"
# Returns "Alice", sets name to "Bob"

# GETDEL: get value and delete the key
GETDEL name
# Returns "Bob", deletes the key

# GETEX: get value and set a new TTL (Redis 6.2+)
GETEX session_token EX 7200   # refreshes TTL to 2 hours
GETEX session_token PERSIST   # remove TTL
GETEX session_token EXAT 1800000000
```

## SETNX (Legacy)

```bash
# Old way to set only if not exists
SETNX username "alice"
# Returns 1 (set) or 0 (already existed)

# Modern equivalent:
SET username "alice" NX
```

## Full Example in Python

```python
import redis

r = redis.Redis(host='127.0.0.1', port=6379, decode_responses=True)

# Basic set and get
r.set('name', 'Alice')
print(r.get('name'))       # Alice

# Set with TTL
r.set('token', 'abc123', ex=3600)
print(r.ttl('token'))      # ~3600

# Conditional set
result = r.set('lock', '1', nx=True)
print(result)              # True (set succeeded)
result = r.set('lock', '2', nx=True)
print(result)              # None (key already exists)

# Multiple keys
r.mset({'a': '1', 'b': '2', 'c': '3'})
values = r.mget('a', 'b', 'c')
print(values)              # ['1', '2', '3']
```

## Summary

SET and GET are the core of Redis string operations. Use EX for automatic expiration, NX for safe conditional writes (like distributed locks), and MSET/MGET for batch operations. The GET option on SET (Redis 6.2+) lets you swap a value and retrieve the old one atomically.
