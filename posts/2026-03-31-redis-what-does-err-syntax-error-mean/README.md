# What Does "ERR syntax error" Mean in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Error, Debugging, Commands

Description: Understand the Redis "ERR syntax error" message, common causes like wrong argument order or unknown options, and how to fix command syntax issues.

---

The `ERR syntax error` in Redis means a command was issued with arguments that Redis could not parse. This is different from a wrong number of arguments error. It typically means arguments are in the wrong order, an unknown option was passed, or a required keyword is missing.

## What Causes the Error

```bash
# Missing EX keyword
127.0.0.1:6379> SET mykey "hello" 300
(error) ERR syntax error

# Unknown option
127.0.0.1:6379> SET mykey "hello" TTL 300
(error) ERR syntax error

# Wrong argument order
127.0.0.1:6379> ZADD leaderboard GT 100 player1 WITHSCORES
(error) ERR syntax error
```

## Common Commands Where This Appears

**SET with options:**

```bash
# Correct syntax
127.0.0.1:6379> SET mykey "hello" EX 300
OK
127.0.0.1:6379> SET mykey "hello" PX 300000
OK
127.0.0.1:6379> SET mykey "hello" EX 300 NX
OK
127.0.0.1:6379> SET mykey "hello" KEEPTTL
OK
```

**ZADD with flags:**

```bash
# Correct syntax
127.0.0.1:6379> ZADD leaderboard 100 player1
(integer) 1
127.0.0.1:6379> ZADD leaderboard GT 200 player1
(integer) 0
127.0.0.1:6379> ZADD leaderboard NX 150 player2
(integer) 1
```

**EXPIRE with conditions (Redis 7+):**

```bash
# Correct syntax
127.0.0.1:6379> EXPIRE mykey 60 NX
(integer) 1
127.0.0.1:6379> EXPIRE mykey 60 XX
(integer) 0
127.0.0.1:6379> EXPIRE mykey 60 GT
(integer) 0
```

## Diagnosing the Issue

Use `COMMAND DOCS` (Redis 7+) to look up exact command syntax:

```bash
127.0.0.1:6379> COMMAND DOCS SET
```

Or consult the Redis CLI help:

```bash
127.0.0.1:6379> HELP SET
```

## Examples of Correct vs Incorrect Syntax

```bash
# Wrong: no EX keyword
SET key value 60
# Right: use EX keyword
SET key value EX 60

# Wrong: NX before EX
SET key value NX EX 60
# Right: EX before or after NX works, but check docs
SET key value EX 60 NX

# Wrong: WITHSCORES on ZADD (it is for ZRANGE, not ZADD)
ZADD myset 1.0 member WITHSCORES
# Right: WITHSCORES applies to read commands
ZRANGE myset 0 -1 WITHSCORES
```

## Handling in Application Code

```python
import redis

r = redis.Redis(decode_responses=True)

try:
    r.execute_command('SET', 'mykey', 'hello', 'TTL', 300)
except redis.exceptions.ResponseError as e:
    if 'syntax error' in str(e).lower():
        print("Command syntax is wrong. Check argument names and order.")
        # Correct call
        r.set('mykey', 'hello', ex=300)
```

## Version-Specific Syntax Changes

Some Redis options were introduced in newer versions. Calling them on older Redis returns ERR syntax error:

```bash
# GETDEL was added in Redis 6.2
127.0.0.1:6379> GETDEL mykey
(error) ERR syntax error  # If running Redis < 6.2
```

Check your Redis version:

```bash
127.0.0.1:6379> INFO server | grep redis_version
```

## Summary

The "ERR syntax error" in Redis means the command arguments could not be parsed due to wrong keywords, incorrect order, or unknown options. Consult the Redis documentation or use `HELP <command>` in the CLI to verify the exact syntax, and check your Redis version when using commands with options introduced in newer releases.
