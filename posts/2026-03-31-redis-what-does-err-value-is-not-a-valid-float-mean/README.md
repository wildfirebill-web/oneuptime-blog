# What Does 'ERR value is not a valid float' Mean in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Error, Debugging, Data Types

Description: Understand the Redis 'ERR value is not a valid float' error, which commands trigger it, and how to validate numeric inputs before sending to Redis.

---

The `ERR value is not a valid float` error in Redis occurs when a command that expects a floating-point number receives a value that cannot be parsed as one. This commonly happens with `INCRBYFLOAT`, `ZADD`, and `ZINCRBY`.

## What Causes the Error

```bash
127.0.0.1:6379> SET counter "hello"
OK
127.0.0.1:6379> INCRBYFLOAT counter 1.5
(error) ERR value is not a valid float
```

```bash
127.0.0.1:6379> ZADD leaderboard notanumber player1
(error) ERR value is not a valid float
```

The error occurs because:
- The key holds a non-numeric string (as in the first example)
- You pass a non-numeric argument where a float is expected (as in the second example)

## Commands That Can Trigger This Error

- `INCRBYFLOAT key increment` - both the stored value and the increment must be valid floats
- `ZADD key score member` - the score must be a valid float
- `ZINCRBY key increment member` - the increment must be a valid float

## Inspecting the Current Value

```bash
127.0.0.1:6379> GET counter
"hello"
127.0.0.1:6379> TYPE counter
string
```

## Fixing the Error

**Reset the key to a valid numeric value:**

```bash
127.0.0.1:6379> SET counter "0"
OK
127.0.0.1:6379> INCRBYFLOAT counter 1.5
"1.5"
```

**Validate before calling Redis:**

```python
import redis

r = redis.Redis(decode_responses=True)

def safe_increment(key, amount):
    try:
        amount = float(amount)
    except (ValueError, TypeError):
        raise ValueError(f"Amount must be a number, got: {amount}")

    current = r.get(key)
    if current is not None:
        try:
            float(current)
        except ValueError:
            raise ValueError(f"Key {key} holds non-numeric value: {current}")

    return r.incrbyfloat(key, amount)
```

## Valid and Invalid Float Formats

Redis accepts these float formats:

```bash
127.0.0.1:6379> SET num "3.14"
OK
127.0.0.1:6379> INCRBYFLOAT num 0
"3.14"  # Valid

127.0.0.1:6379> SET num "1e3"
OK
127.0.0.1:6379> INCRBYFLOAT num 0
"1000"  # Scientific notation is valid

127.0.0.1:6379> SET num "inf"
OK
127.0.0.1:6379> INCRBYFLOAT num 0
"inf"   # Infinity is valid for ZADD scores
```

These are invalid:

```text
"3.14abc"   - trailing non-numeric characters
"hello"     - non-numeric string
""          - empty string
"NaN"       - not a number
```

## Validating ZADD Scores in Application Code

```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function addToLeaderboard(member, score) {
  if (typeof score !== 'number' || !isFinite(score)) {
    throw new Error(`Invalid score: ${score}`);
  }
  await redis.zadd('leaderboard', score, member);
}
```

## Summary

The "ERR value is not a valid float" error means Redis received a non-numeric value where a float was required. Validate user inputs before calling `INCRBYFLOAT`, `ZADD`, or `ZINCRBY`, ensure stored string values are numeric before incrementing, and use `GET` to inspect the current stored value when debugging.
