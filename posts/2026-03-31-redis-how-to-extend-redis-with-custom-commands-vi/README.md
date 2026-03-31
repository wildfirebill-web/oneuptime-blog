# How to Extend Redis with Custom Commands via Modules

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Modules, Custom Commands, Lua, Extensions, Advanced, API

Description: Learn multiple approaches to extending Redis with custom commands, including Lua scripts, Redis modules in C, and the module development best practices.

---

## Ways to Extend Redis

Redis can be extended in several ways:

```text
Approach            Language    Use Case
--------            --------    --------
Lua scripts         Lua         Simple atomic operations, available in all Redis versions
Redis Modules       C           New data types, complex commands, high performance
Redis Functions     Lua/JS      Modern scripting (Redis 7+), persistent named functions
```

## Approach 1: Lua Scripts for Custom Logic

Lua scripts run atomically inside Redis. No other commands execute while a script runs.

```bash
# Run a Lua script inline
redis-cli EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 mykey myvalue

# Multi-step atomic operation
redis-cli EVAL "
  local current = redis.call('GET', KEYS[1])
  if current == false then
    redis.call('SET', KEYS[1], ARGV[1])
    return 'initialized'
  else
    return 'already set: ' .. current
  end
" 1 mykey "hello"
```

### Named Lua Scripts with SCRIPT LOAD

```bash
# Load a script and get its SHA1 hash
SHA=$(redis-cli SCRIPT LOAD "
  local val = redis.call('INCR', KEYS[1])
  if val > tonumber(ARGV[1]) then
    redis.call('SET', KEYS[1], ARGV[1])
    return tonumber(ARGV[1])
  end
  return val
")
echo "Script SHA: $SHA"

# Execute using the SHA (faster - no script transmission)
redis-cli EVALSHA $SHA 1 mycounter 100
```

### Node.js Lua Script Example

```javascript
const Redis = require('ioredis');
const redis = new Redis({ host: 'localhost', port: 6379 });

// Define a custom "rate limit with reset" command via Lua
const rateLimitScript = `
local key = KEYS[1]
local max = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call('GET', key)
if current == false then
  redis.call('SET', key, 1)
  redis.call('EXPIRE', key, window)
  return {1, max, window}
end

local count = tonumber(current)
if count >= max then
  local ttl = redis.call('TTL', key)
  return {count, max, ttl}
end

redis.call('INCR', key)
local ttl = redis.call('TTL', key)
return {count + 1, max, ttl}
`;

async function checkRateLimit(identifier, maxRequests, windowSeconds) {
  const key = `rate:${identifier}`;
  const [current, max, ttl] = await redis.eval(rateLimitScript, 1, key, maxRequests, windowSeconds);

  return {
    allowed: current <= max,
    current: parseInt(current),
    limit: parseInt(max),
    resetIn: parseInt(ttl)
  };
}

// Usage
const result = await checkRateLimit('user:42:api', 100, 60);
if (!result.allowed) {
  console.log(`Rate limited. Resets in ${result.resetIn} seconds`);
}
```

## Approach 2: Redis Functions (Redis 7+)

Redis Functions are persistent named scripts stored in Redis (unlike EVAL which is stateless):

```bash
# Define a function library
redis-cli FUNCTION LOAD "#!lua name=mylib
  local function increment_with_cap(keys, args)
    local key = keys[1]
    local increment = tonumber(args[1])
    local cap = tonumber(args[2])

    local current = redis.call('INCR', key)
    if current > cap then
      redis.call('SET', key, cap)
      return cap
    end
    return current
  end

  redis.register_function('incrwithcap', increment_with_cap)
"

# Call the function
redis-cli FCALL incrwithcap 1 mycounter 5 100
```

## Approach 3: Redis Module with New Data Type

A more advanced module that creates a custom data type for a bounded counter:

```c
#include "redismodule.h"

/* Custom data type structure */
typedef struct BoundedCounter {
    long long value;
    long long min;
    long long max;
} BoundedCounter;

/* Create a new bounded counter */
int BCounter_Create_RedisCommand(RedisModuleCtx *ctx, RedisModuleString **argv, int argc) {
    if (argc != 4) return RedisModule_WrongArity(ctx);

    RedisModule_AutoMemory(ctx);

    long long min, max;
    if (RedisModule_StringToLongLong(argv[2], &min) != REDISMODULE_OK ||
        RedisModule_StringToLongLong(argv[3], &max) != REDISMODULE_OK) {
        return RedisModule_ReplyWithError(ctx, "ERR invalid min/max");
    }

    if (min >= max) {
        return RedisModule_ReplyWithError(ctx, "ERR min must be less than max");
    }

    BoundedCounter *bc = RedisModule_Alloc(sizeof(BoundedCounter));
    bc->value = min;
    bc->min = min;
    bc->max = max;

    /* Store the counter using a module-specific type */
    /* (simplified - full implementation needs type registration) */
    RedisModuleKey *key = RedisModule_OpenKey(ctx, argv[1], REDISMODULE_WRITE);
    RedisModule_ModuleTypeSetValue(key, NULL /* type */, bc);

    return RedisModule_ReplyWithOk(ctx);
}
```

## Registering Multiple Commands in a Module

```c
int RedisModule_OnLoad(RedisModuleCtx *ctx, RedisModuleString **argv, int argc) {
    if (RedisModule_Init(ctx, "myext", 1, REDISMODULE_APIVER_1) == REDISMODULE_ERR) {
        return REDISMODULE_ERR;
    }

    /* Register multiple commands */
    struct {
        char *name;
        RedisModuleCmdFunc func;
        char *flags;
        int firstkey;
        int lastkey;
        int step;
    } commands[] = {
        {"myext.create", MyExt_Create, "write fast", 1, 1, 1},
        {"myext.get",    MyExt_Get,    "readonly fast", 1, 1, 1},
        {"myext.incr",   MyExt_Incr,   "write fast", 1, 1, 1},
        {"myext.reset",  MyExt_Reset,  "write fast", 1, 1, 1},
        {NULL}
    };

    for (int i = 0; commands[i].name; i++) {
        if (RedisModule_CreateCommand(ctx,
                commands[i].name,
                commands[i].func,
                commands[i].flags,
                commands[i].firstkey,
                commands[i].lastkey,
                commands[i].step) == REDISMODULE_ERR) {
            return REDISMODULE_ERR;
        }
    }

    return REDISMODULE_OK;
}
```

## Python: Registering and Using Custom Scripts

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Register a script with a cached SHA
class CustomRedis:
    def __init__(self, client):
        self.client = client
        self._scripts = {}

    def register_script(self, name: str, lua_code: str):
        sha = self.client.script_load(lua_code)
        self._scripts[name] = sha
        return sha

    def call_script(self, name: str, keys: list, args: list):
        sha = self._scripts.get(name)
        if not sha:
            raise KeyError(f"Script '{name}' not registered")
        return self.client.evalsha(sha, len(keys), *keys, *args)

# Usage
redis_ext = CustomRedis(r)

redis_ext.register_script('safe_increment', """
local current = tonumber(redis.call('GET', KEYS[1])) or 0
local new_val = current + tonumber(ARGV[1])
if new_val > tonumber(ARGV[2]) then
  return redis.error_reply('ERR exceeds maximum')
end
redis.call('SET', KEYS[1], new_val)
return new_val
""")

result = redis_ext.call_script('safe_increment', ['my-counter'], [5, 100])
print(f"New value: {result}")
```

## Module Command Flags Reference

```text
Flag            Meaning
----            -------
write           Command modifies keys
readonly        Command does not modify keys
admin           Administrative command
fast            O(1) or O(log N) complexity
no-cluster      Not supported in cluster mode
deny-oom        Reject when out of memory
```

## Summary

Redis can be extended via Lua scripts for atomic custom logic, Redis Functions (Redis 7+) for persistent named scripts, and C modules for new data types and high-performance commands. Lua scripts are the lowest barrier to entry and work in all Redis versions. Redis Functions improve on EVAL by making scripts persistent and named. C modules provide the deepest integration but require C knowledge and module API familiarity. Choose the approach that matches your complexity and performance needs.
