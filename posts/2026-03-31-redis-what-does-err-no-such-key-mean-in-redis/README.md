# What Does 'ERR no such key' Mean in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Error, ERR No Such Key, Key Expiry, Debugging, Troubleshooting

Description: Understand when Redis returns 'ERR no such key', which commands trigger it, and how to handle missing keys gracefully in your application code.

---

## What Is the "ERR no such key" Error

When you run certain Redis commands on a key that does not exist, Redis returns:

```text
(error) ERR no such key
```

This is different from commands like `GET` or `EXISTS` which return `nil` or `0` for missing keys. Only a small subset of commands return this error for missing keys.

## Which Commands Return This Error

Most Redis commands return `nil` or `0` for missing keys. The main commands that return `ERR no such key` are:

```bash
# OBJECT - inspecting encoding, idletime, refcount, frequency
redis-cli OBJECT ENCODING missing-key
# (error) ERR no such key

redis-cli OBJECT IDLETIME missing-key
# (error) ERR no such key

redis-cli OBJECT REFCOUNT missing-key
# (error) ERR no such key

redis-cli OBJECT FREQ missing-key
# (error) ERR no such key

# OBJECT HELP lists available subcommands (does not need a key)
redis-cli OBJECT HELP
```

Other commands that may return this error in certain contexts:

```bash
# DUMP on a non-existent key
redis-cli DUMP missing-key
# (nil) - returns nil, not an error

# RENAME when the source key does not exist
redis-cli RENAME missing-key new-key
# (error) ERR no such key

# RENAMENX when the source key does not exist
redis-cli RENAMENX missing-key new-key
# (error) ERR no such key
```

## Common Causes

### Key Never Existed

The key was not set before you tried to access it.

### Key Expired

The key had a TTL set and has since expired:

```bash
redis-cli SET temp-key "value" EX 10
# Wait 11 seconds...
redis-cli OBJECT ENCODING temp-key
# (error) ERR no such key
```

### Keyspace Notifications Delivery Delay

The key existed when you received a notification but expired before you could inspect it.

### RENAME on Missing Source

```bash
redis-cli RENAME source-key destination-key
# (error) ERR no such key - if source-key does not exist
```

## How to Handle Missing Keys

### Check Existence First

```bash
# For OBJECT commands
redis-cli EXISTS mykey
# (integer) 1  -> key exists, safe to call OBJECT
# (integer) 0  -> key does not exist
```

### Use GET Instead of OBJECT for Type Checking

For most use cases, you do not need `OBJECT ENCODING`. Use `TYPE` instead:

```bash
redis-cli TYPE mykey
# string / list / set / zset / hash / stream / none
```

`TYPE` returns `none` for missing keys instead of an error.

### Handle RENAME Safely with Transactions

```python
import redis

r = redis.Redis()

def safe_rename(src, dst):
    if not r.exists(src):
        return False
    r.rename(src, dst)
    return True
```

Or use WATCH to handle race conditions:

```python
def atomic_rename(r, src, dst):
    with r.pipeline() as pipe:
        while True:
            try:
                pipe.watch(src)
                if not pipe.exists(src):
                    pipe.reset()
                    return False
                pipe.multi()
                pipe.rename(src, dst)
                pipe.execute()
                return True
            except redis.WatchError:
                continue
```

## Handling in Application Code

### Python

```python
import redis

r = redis.Redis()

def get_key_encoding(key):
    try:
        return r.object_encoding(key)
    except redis.exceptions.ResponseError as e:
        if 'no such key' in str(e):
            return None  # Key does not exist
        raise

def safe_rename(src, dst):
    try:
        return r.rename(src, dst)
    except redis.exceptions.ResponseError as e:
        if 'no such key' in str(e):
            return False
        raise
```

### Node.js

```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function getKeyEncoding(key) {
  try {
    return await redis.object('encoding', key);
  } catch (err) {
    if (err.message.includes('ERR no such key')) {
      return null;
    }
    throw err;
  }
}

async function safeRename(src, dst) {
  const exists = await redis.exists(src);
  if (!exists) return false;
  await redis.rename(src, dst);
  return true;
}
```

## Distinguishing Between Key Types for Missing Keys

```bash
# These return nil (not an error) for missing keys:
redis-cli GET missing-key        # nil
redis-cli HGET missing-key field  # nil
redis-cli LRANGE missing-key 0 -1 # (empty array)
redis-cli SMEMBERS missing-key    # (empty array)
redis-cli TTL missing-key         # -2 (key does not exist)
redis-cli TYPE missing-key        # none

# These return errors for missing keys:
redis-cli OBJECT ENCODING missing-key  # ERR no such key
redis-cli RENAME missing-key other-key  # ERR no such key
```

## Debugging Key Expiry Issues

If you are getting unexpected `ERR no such key` for keys you believe should exist, check if they have TTLs:

```bash
# Check TTL of a key
redis-cli TTL mykey
# -1 = no expiry
# -2 = does not exist
# N  = seconds until expiry

# Check in PTTL (milliseconds)
redis-cli PTTL mykey
```

Enable keyspace notifications to monitor key expiry events:

```bash
redis-cli CONFIG SET notify-keyspace-events Ex
redis-cli SUBSCRIBE "__keyevent@0__:expired"
```

## Summary

The "ERR no such key" error is returned by `OBJECT` subcommands and `RENAME`/`RENAMENX` when the specified key does not exist. For most commands, a missing key returns `nil` or `0` instead. Handle it by checking key existence before calling `OBJECT` commands, using the `TYPE` command instead of `OBJECT ENCODING` when the key might not exist, and implementing existence checks before `RENAME` operations.
