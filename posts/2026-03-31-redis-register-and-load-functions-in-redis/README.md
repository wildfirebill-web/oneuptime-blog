# How to Register and Load Functions in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Function, Library, Lua, Server-Side Logic

Description: Learn how to register and load Redis function libraries using FUNCTION LOAD, making reusable server-side logic available across client connections.

---

Redis Functions (introduced in Redis 7.0) let you load named libraries of server-side functions that persist across restarts. Unlike EVAL scripts, functions have names, belong to libraries, and are stored durably in Redis.

## Understanding Redis Functions vs Lua Scripts

| Feature | EVAL Scripts | Redis Functions |
|---|---|---|
| Named | No (referenced by SHA) | Yes (library + function name) |
| Persistent | No (lost on restart) | Yes (persists in RDB/AOF) |
| Replication | Per call | Library replicates to replicas |
| Namespace | None | Library-scoped |

## Create a Function Library

Write a Lua library file:

```lua
-- mylib.lua
#!lua name=mylib

-- Register a function to increment a counter with expiry
redis.register_function('increment_with_ttl', function(keys, args)
    local key = keys[1]
    local ttl = tonumber(args[1]) or 3600
    local amount = tonumber(args[2]) or 1

    local new_val = redis.call('INCRBY', key, amount)
    if new_val == amount then
        -- Key was just created, set TTL
        redis.call('EXPIRE', key, ttl)
    end
    return new_val
end)

-- Register a function to set a value only if not exists, with TTL
redis.register_function('setnx_with_ttl', function(keys, args)
    local key = keys[1]
    local value = args[1]
    local ttl = tonumber(args[2]) or 60

    local result = redis.call('SET', key, value, 'NX', 'EX', ttl)
    if result then
        return 1
    end
    return 0
end)
```

## Load the Library

```bash
redis-cli FUNCTION LOAD "$(cat mylib.lua)"
# Returns: mylib
```

If the library already exists, use `REPLACE` to update it:

```bash
redis-cli FUNCTION LOAD REPLACE "$(cat mylib.lua)"
```

## Verify the Library is Loaded

```bash
# List all loaded libraries
redis-cli FUNCTION LIST

# List with code
redis-cli FUNCTION LIST WITHCODE

# Show a specific library
redis-cli FUNCTION LIST LIBRARYNAME mylib
```

Sample output:

```text
1) 1) "library_name"
   2) "mylib"
   3) "engine"
   4) "LUA"
   5) "functions"
   6) 1) 1) "name"
         2) "increment_with_ttl"
         ...
```

## Call the Functions

Use `FCALL` to invoke a registered function:

```bash
# Increment counter with 1-hour TTL, by 1
redis-cli FCALL increment_with_ttl 1 my-counter 3600 1

# Set a key only if not exists with 60s TTL
redis-cli FCALL setnx_with_ttl 1 my-lock "token123" 60
```

## Load from Python

```python
import redis

r = redis.Redis()

with open("mylib.lua") as f:
    library_code = f.read()

# Load or replace the library
try:
    r.function_load(library_code)
except redis.exceptions.ResponseError as e:
    if "already exists" in str(e):
        r.function_load(library_code, replace=True)
    else:
        raise

# Call the function
result = r.fcall("increment_with_ttl", 1, "my-counter", 3600, 5)
print(f"New value: {result}")
```

## Persistence

Redis Functions persist automatically in RDB and AOF files. After a restart, they are available immediately. Verify:

```bash
redis-cli DEBUG RELOAD
redis-cli FUNCTION LIST
# Library should still be present
```

## Summary

Redis Functions are loaded with `FUNCTION LOAD` using a library file that declares a name and registers named functions with `redis.register_function()`. Functions persist in Redis storage (RDB/AOF), replicate to replicas, and are called by name with `FCALL`. Use `FUNCTION LOAD REPLACE` to update existing libraries. This is the modern alternative to EVAL-based scripting for production server-side logic.
