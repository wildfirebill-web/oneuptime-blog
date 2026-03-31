# How to Return Multiple Values from Redis Lua Scripts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Lua, Scripting, Return Value, Array

Description: Learn how to return multiple values from Redis Lua scripts using Lua arrays and how to parse them in Python, Node.js, and Go clients.

---

Redis Lua scripts can return a single value, but that value can be a Lua array table containing multiple items. Understanding how return types map between Lua and Redis lets you return rich structured data from a single script call.

## Return Type Mapping

| Lua Type | Redis Reply | Client Type |
|---|---|---|
| Number | Integer | integer |
| String | Bulk string | string |
| `false` or `nil` | Nil | nil/null |
| Table (array) | Array | array/list |
| `redis.status_reply("OK")` | Status | status string |
| `redis.error_reply("msg")` | Error | exception |

## Returning an Array

```lua
-- Return multiple values as an array
local val1 = redis.call('GET', KEYS[1])
local val2 = redis.call('HGET', KEYS[2], ARGV[1])
local count = redis.call('LLEN', KEYS[3])

return {val1, val2, count}
```

```bash
redis-cli EVAL "$(cat multi_return.lua)" 3 key1 myhash mylist field1
# Returns:
# 1) "value1"
# 2) "hashvalue"
# 3) (integer) 42
```

## Mixed Type Arrays

Arrays can contain strings, integers, and even nested arrays:

```lua
local results = {}

-- Add integer
table.insert(results, redis.call('INCR', KEYS[1]))

-- Add string
table.insert(results, redis.call('GET', KEYS[2]))

-- Add boolean-like value
local exists = redis.call('EXISTS', KEYS[3])
table.insert(results, exists)

return results
```

## Returning Named Pairs (Flattened)

Redis cannot return nested tables as dictionaries. Use flat key-value pairs:

```lua
-- Return user data as flat array: [field, value, field, value, ...]
local fields = {'name', 'email', 'score', 'rank'}
local result = {}

for _, field in ipairs(fields) do
    table.insert(result, field)
    table.insert(result, redis.call('HGET', KEYS[1], field))
end

return result
```

In Python, reconstruct the dictionary:

```python
raw = r.eval(script, 1, "user:42")
user = dict(zip(raw[::2], raw[1::2]))
# {"name": "Alice", "email": "alice@...", "score": "100", "rank": "3"}
```

## Parsing Multi-Returns in Node.js

```javascript
const result = await redis.eval(script, 1, "user:42");
// result is an array
const [value1, value2, count] = result;
```

## Parsing in Go with go-redis

```go
result, err := rdb.Eval(ctx, script, []string{"user:42"}).Result()
if err != nil {
    log.Fatal(err)
}
values := result.([]interface{})
fmt.Println(values[0], values[1])
```

## Returning Status and Data Together

Return a status flag with the data:

```lua
-- Return [status, value] where status is 1=found 0=not-found
local val = redis.call('GET', KEYS[1])
if val == false then
    return {0, false}
end
return {1, val}
```

## Nested Arrays

Lua can return nested arrays (arrays within arrays):

```lua
-- Return multiple lists
local list1 = redis.call('LRANGE', KEYS[1], 0, 4)
local list2 = redis.call('LRANGE', KEYS[2], 0, 4)

return {list1, list2}
```

```bash
redis-cli EVAL "$(cat nested.lua)" 2 listA listB
# Returns:
# 1) 1) "a1"
#    2) "a2"
# 2) 1) "b1"
#    2) "b2"
```

## Summary

Return multiple values from Redis Lua scripts by building a Lua array table and returning it. Lua arrays become Redis multi-bulk replies (arrays). Nested arrays are supported for hierarchical data. For dictionary-like data, flatten to key-value pairs and reconstruct in the client. Check the Redis client documentation to understand how it maps multi-bulk replies to native language types (list, array, slice).
