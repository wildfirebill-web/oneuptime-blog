# How to Migrate from Lua Scripts to Redis Functions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Function, Lua, Migration, Modernization

Description: Migrate your Redis EVAL Lua scripts to named Redis Functions for better organization, persistence, and maintenance in Redis 7 and later.

---

Redis Functions (available in Redis 7.0+) are the modern successor to EVAL-based Lua scripting. They provide named, persistent, replicable server-side logic. Migrating existing scripts to functions improves code organization and eliminates the need to manage script SHAs.

## Comparing EVAL Scripts and Functions

An EVAL script:

```python
# Application manages the script text and SHA
RATE_LIMIT_SCRIPT = """
local key = KEYS[1]
local limit = tonumber(ARGV[1])
...
"""
sha = r.script_load(RATE_LIMIT_SCRIPT)
result = r.evalsha(sha, 1, "ratelimit:user:42", 100)
```

The same logic as a Redis Function:

```lua
-- ratelimit_lib.lua
#!lua name=ratelimit

redis.register_function('check_rate_limit', function(keys, args)
    local key = keys[1]
    local limit = tonumber(args[1])
    -- ... same logic
end)
```

```python
# Application calls by name - no SHA management
result = r.fcall("check_rate_limit", 1, "ratelimit:user:42", 100)
```

## Step-by-Step Migration

### Step 1 - Create a Library File

Wrap your script in a library declaration:

```lua
-- Before (EVAL script stored in a Python file)
"""
local current = redis.call('INCR', KEYS[1])
if current == 1 then
    redis.call('EXPIRE', KEYS[1], tonumber(ARGV[1]))
end
if current > tonumber(ARGV[2]) then
    return 0
end
return 1
"""
```

```lua
-- After: functions/rate_limit.lua
#!lua name=rate_limit

redis.register_function('check_rate_limit', function(keys, args)
    local key = keys[1]
    local window_seconds = tonumber(args[1])
    local max_requests = tonumber(args[2])

    local current = redis.call('INCR', key)
    if current == 1 then
        redis.call('EXPIRE', key, window_seconds)
    end
    if current > max_requests then
        return 0
    end
    return 1
end)
```

### Step 2 - Load the Library

```bash
redis-cli FUNCTION LOAD "$(cat functions/rate_limit.lua)"
```

### Step 3 - Update Application Code

```python
# Before - using EVAL/EVALSHA
def check_rate_limit(user_id, window=60, max_req=100):
    key = f"ratelimit:{user_id}"
    return r.evalsha(RATE_LIMIT_SHA, 1, key, window, max_req)

# After - using FCALL
def check_rate_limit(user_id, window=60, max_req=100):
    key = f"ratelimit:{user_id}"
    return r.fcall("check_rate_limit", 1, key, window, max_req)
```

### Step 4 - Handle the Transition Period

Run both code paths simultaneously during rollout:

```python
USE_FUNCTIONS = os.getenv("USE_REDIS_FUNCTIONS", "false") == "true"

def check_rate_limit(user_id, window=60, max_req=100):
    key = f"ratelimit:{user_id}"
    if USE_FUNCTIONS:
        return r.fcall("check_rate_limit", 1, key, window, max_req)
    else:
        return r.evalsha(RATE_LIMIT_SHA, 1, key, window, max_req)
```

## Key Differences to Handle

**Key and arg access:** In EVAL, keys are `KEYS[1]` (1-indexed Lua). In functions, keys are `keys[1]` (still 1-indexed but passed as a Lua array to your function):

```lua
-- EVAL style
local key = KEYS[1]

-- Function style
redis.register_function('my_func', function(keys, args)
    local key = keys[1]   -- Same index, different variable name
    local arg1 = args[1]
end)
```

**Flags for read-only functions:** Mark functions as read-only to run them on replicas:

```lua
redis.register_function{
    function_name = 'get_stats',
    callback = function(keys, args)
        return redis.call('GET', keys[1])
    end,
    flags = {'no-writes'}
}
```

## Verify Migration

```bash
# Confirm function is loaded
redis-cli FUNCTION LIST LIBRARYNAME rate_limit

# Test with FCALL
redis-cli FCALL check_rate_limit 1 ratelimit:test-user 60 100
```

## Summary

Migrating from EVAL scripts to Redis Functions requires wrapping your Lua logic in a library file with `#!lua name=libname` and using `redis.register_function()`. Load with `FUNCTION LOAD`, call with `FCALL`, and update application code to reference functions by name instead of managing SHAs. Use a feature flag for gradual rollout and verify with `FUNCTION LIST` after deployment.
