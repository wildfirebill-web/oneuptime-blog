# What Does 'ERR value is not an integer' Mean in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Error, Debugging, Data Types

Description: Understand the Redis 'ERR value is not an integer or out of range' error, which commands cause it, and how to validate integer inputs in your application.

---

The `ERR value is not an integer or out of range` error appears when a Redis command that requires an integer receives a non-integer value. This most often occurs with counter operations and expiration commands.

## What Causes the Error

```bash
127.0.0.1:6379> SET counter "hello"
OK
127.0.0.1:6379> INCR counter
(error) ERR value is not an integer or out of range
```

```bash
127.0.0.1:6379> EXPIRE mykey notanumber
(error) ERR value is not an integer or out of range
```

```bash
127.0.0.1:6379> SET counter "3.14"
OK
127.0.0.1:6379> INCR counter
(error) ERR value is not an integer or out of range
```

Floats stored as strings also fail integer operations because `INCR` requires a strictly integer value.

## Commands That Trigger This Error

- `INCR key` and `INCRBY key increment`
- `DECR key` and `DECRBY key decrement`
- `EXPIRE key seconds` and `PEXPIRE key milliseconds`
- `EXPIREAT key timestamp` and `PEXPIREAT key timestamp`
- `SETRANGE key offset value`
- `GETRANGE key start end`
- `LINDEX key index`
- `LINSERT` position arguments

## Inspecting the Stored Value

```bash
127.0.0.1:6379> GET counter
"hello"
127.0.0.1:6379> TYPE counter
string
```

## Fixing the Error

**Reset to a valid integer:**

```bash
127.0.0.1:6379> SET counter "0"
OK
127.0.0.1:6379> INCR counter
(integer) 1
```

**Validate before calling Redis in Python:**

```python
import redis

r = redis.Redis(decode_responses=True)

def safe_increment(key, by=1):
    current = r.get(key)
    if current is not None:
        try:
            int(current)
        except ValueError:
            raise ValueError(f"Key '{key}' contains non-integer value: '{current}'")

    by = int(by)  # Will raise ValueError if not an integer
    return r.incrby(key, by)
```

## Out of Range Case

Redis integers are signed 64-bit. Incrementing beyond `9223372036854775807` also triggers this error:

```bash
127.0.0.1:6379> SET counter 9223372036854775807
OK
127.0.0.1:6379> INCR counter
(error) ERR increment or decrement would overflow
```

## Fixing Expiration Errors

If the error occurs on `EXPIRE`, ensure you pass a whole number:

```python
import math

ttl_seconds = 3600.5  # Floating TTL from a calculation

# Fix by rounding
r.expire("mykey", math.floor(ttl_seconds))

# Or use PEXPIRE for millisecond precision
r.pexpire("mykey", int(ttl_seconds * 1000))
```

## Atomic Initialization Pattern

Use `SET NX` to safely initialize a counter before incrementing:

```bash
127.0.0.1:6379> SET counter 0 NX
OK
127.0.0.1:6379> INCR counter
(integer) 1
```

In Python:

```python
r.set('counter', 0, nx=True)  # Only sets if key does not exist
r.incr('counter')
```

## Summary

The "ERR value is not an integer or out of range" error means the stored value or an argument is not a valid 64-bit integer. Always validate that counter keys contain only whole numbers, ensure expiration arguments are integers, and use `PEXPIRE` when millisecond precision requires fractional seconds.
