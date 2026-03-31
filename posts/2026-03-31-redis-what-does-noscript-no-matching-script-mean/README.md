# What Does "NOSCRIPT No matching script" Mean in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Lua, Scripting, Error, Debugging

Description: Understand the Redis NOSCRIPT error, when it occurs with EVALSHA, and how to handle script caching failures gracefully in production.

---

The `NOSCRIPT No matching script. Please use EVAL.` error occurs when you call `EVALSHA` with a script SHA1 hash that Redis does not have in its script cache. Understanding when this happens and how to handle it is important for production reliability.

## What Causes the Error

Redis caches Lua scripts server-side using `SCRIPT LOAD` or after the first `EVAL` call. When you call `EVALSHA`, Redis looks up the script by its SHA1 hash. If the script is not in the cache, you get NOSCRIPT.

This commonly happens after:
- A Redis server restart (the script cache is not persisted)
- A `SCRIPT FLUSH` command
- Connecting to a different replica or cluster node that never received the script

```bash
127.0.0.1:6379> EVALSHA abc123deadbeef 0
(error) NOSCRIPT No matching script. Please use EVAL.
```

## Loading a Script into the Cache

Use `SCRIPT LOAD` to pre-load a script and get its SHA1 hash:

```bash
127.0.0.1:6379> SCRIPT LOAD "return redis.call('GET', KEYS[1])"
"d66eef6b2b430a0e4e59591fc85a3e3f1d5a94f3"
```

Then call it with `EVALSHA`:

```bash
127.0.0.1:6379> EVALSHA d66eef6b2b430a0e4e59591fc85a3e3f1d5a94f3 1 mykey
"myvalue"
```

## Handling NOSCRIPT Gracefully

The standard pattern is to try `EVALSHA` first and fall back to `EVAL` if you receive NOSCRIPT.

```python
import redis

r = redis.Redis(decode_responses=True)

SCRIPT = """
local current = redis.call('GET', KEYS[1])
if current then
    return redis.call('INCR', KEYS[1])
end
redis.call('SET', KEYS[1], 1)
return 1
"""

SCRIPT_SHA = r.script_load(SCRIPT)

def safe_evalsha(keys, args):
    try:
        return r.evalsha(SCRIPT_SHA, len(keys), *keys, *args)
    except redis.exceptions.NoScriptError:
        # Reload and retry
        global SCRIPT_SHA
        SCRIPT_SHA = r.script_load(SCRIPT)
        return r.evalsha(SCRIPT_SHA, len(keys), *keys, *args)
```

## Node.js Example with ioredis

```javascript
const Redis = require('ioredis');
const redis = new Redis();

const script = `
  local val = redis.call('GET', KEYS[1])
  if val then return tonumber(val) + 1 end
  redis.call('SET', KEYS[1], 1)
  return 1
`;

let sha;

async function runScript(key) {
  if (!sha) {
    sha = await redis.script('LOAD', script);
  }
  try {
    return await redis.evalsha(sha, 1, key);
  } catch (err) {
    if (err.message.includes('NOSCRIPT')) {
      sha = await redis.script('LOAD', script);
      return redis.evalsha(sha, 1, key);
    }
    throw err;
  }
}
```

## Checking Cached Scripts

```bash
127.0.0.1:6379> SCRIPT EXISTS d66eef6b2b430a0e4e59591fc85a3e3f1d5a94f3
1) (integer) 1

127.0.0.1:6379> SCRIPT EXISTS 0000000000000000000000000000000000000000
1) (integer) 0
```

## Summary

The NOSCRIPT error means Redis does not have the Lua script in its cache, which happens after server restarts or `SCRIPT FLUSH`. Always implement a try-EVALSHA-then-EVAL fallback pattern, and pre-load critical scripts with `SCRIPT LOAD` during application startup to minimize cache misses.
