# How to Use TYPE in Redis to Check a Key's Data Type

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Data Type, Command, Key Management, Debugging

Description: Learn how to use the TYPE command in Redis to identify the data type stored at a key, essential for debugging type errors and building type-aware logic.

---

## What Is TYPE in Redis

`TYPE` returns the data type stored at a key. Redis stores different types - strings, lists, sets, sorted sets, hashes, streams, and more. Knowing the type is essential when you encounter `WRONGTYPE` errors or need to handle keys differently based on their type.

```text
TYPE key
```

Returns a status string: `string`, `list`, `set`, `zset`, `hash`, `stream`, or `none`.

## Return Values

| Return Value | Data Type |
|-------------|-----------|
| `string` | String (also used for numbers and bitmaps) |
| `list` | List |
| `set` | Set |
| `zset` | Sorted set |
| `hash` | Hash |
| `stream` | Stream |
| `none` | Key does not exist |

## Basic Usage

```bash
SET name "Alice"
TYPE name
# string

LPUSH tasks "task1" "task2"
TYPE tasks
# list

SADD colors "red" "green" "blue"
TYPE colors
# set

ZADD scores 100 "alice" 200 "bob"
TYPE scores
# zset

HSET user:1 name "Alice" age 30
TYPE user:1
# hash

XADD events * action "login" user "alice"
TYPE events
# stream

TYPE nonexistent:key
# none
```

## Practical Example in Python

```python
import redis

client = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Create various types
client.set('str_key', 'hello')
client.lpush('list_key', 'a', 'b', 'c')
client.sadd('set_key', 'x', 'y', 'z')
client.zadd('zset_key', {'alice': 100, 'bob': 200})
client.hset('hash_key', mapping={'name': 'Alice', 'age': '30'})

keys = ['str_key', 'list_key', 'set_key', 'zset_key', 'hash_key', 'missing']
for key in keys:
    key_type = client.type(key)
    print(f"{key}: {key_type}")
```

## Practical Example in Node.js

```javascript
const { createClient } = require('redis');

const client = createClient();
await client.connect();

await client.set('myString', 'hello');
await client.lPush('myList', ['a', 'b', 'c']);
await client.sAdd('mySet', ['x', 'y', 'z']);
await client.zAdd('myZset', [{ score: 1, value: 'one' }, { score: 2, value: 'two' }]);
await client.hSet('myHash', { field1: 'value1', field2: 'value2' });

const keys = ['myString', 'myList', 'mySet', 'myZset', 'myHash', 'nonexistent'];
for (const key of keys) {
  const type = await client.type(key);
  console.log(`${key}: ${type}`);
}
```

## Practical Example in Go

```go
package main

import (
    "context"
    "fmt"
    "github.com/redis/go-redis/v9"
)

func main() {
    ctx := context.Background()
    rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})

    keys := []string{"myString", "myList", "mySet", "nonexistent"}
    for _, key := range keys {
        keyType, err := rdb.Type(ctx, key).Result()
        if err != nil {
            fmt.Printf("%s: error - %v\n", key, err)
            continue
        }
        fmt.Printf("%s: %s\n", key, keyType)
    }
}
```

## Type-Aware Key Operations

```python
import redis

client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def inspect_key(key):
    """Get detailed info about a key based on its type."""
    key_type = client.type(key)

    if key_type == 'none':
        return {'key': key, 'type': 'none', 'exists': False}

    info = {'key': key, 'type': key_type, 'exists': True}

    if key_type == 'string':
        info['value'] = client.get(key)
        info['length'] = client.strlen(key)

    elif key_type == 'list':
        info['length'] = client.llen(key)
        info['first_item'] = client.lindex(key, 0)

    elif key_type == 'set':
        info['cardinality'] = client.scard(key)
        info['sample'] = list(client.srandmember(key, 3))

    elif key_type == 'zset':
        info['cardinality'] = client.zcard(key)
        info['top_member'] = client.zrange(key, -1, -1, withscores=True)

    elif key_type == 'hash':
        info['fields'] = client.hlen(key)
        info['all_fields'] = client.hgetall(key)

    elif key_type == 'stream':
        info['length'] = client.xlen(key)
        info['last_entry'] = client.xrevrange(key, count=1)

    return info

# Example usage
client.set('example:string', 'hello world')
client.lpush('example:list', 'item1', 'item2', 'item3')
client.hset('example:hash', mapping={'name': 'test', 'value': '42'})

for key in ['example:string', 'example:list', 'example:hash', 'missing']:
    info = inspect_key(key)
    print(f"\n{key}:")
    for k, v in info.items():
        print(f"  {k}: {v}")
```

## Avoiding WRONGTYPE Errors

The `WRONGTYPE` error occurs when you try to use a command on the wrong data type:

```bash
SET mykey "hello"
LPUSH mykey "item"
# WRONGTYPE Operation against a key holding the wrong kind of value
```

Use `TYPE` to check before operating:

```python
import redis

client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def safe_list_push(key, value):
    """Only push to key if it's a list or doesn't exist."""
    key_type = client.type(key)

    if key_type not in ('list', 'none'):
        raise ValueError(f"Key '{key}' is type '{key_type}', expected 'list' or 'none'")

    return client.rpush(key, value)

try:
    client.set('mykey', 'not a list')
    safe_list_push('mykey', 'item')
except ValueError as e:
    print(f"Type error: {e}")
```

## Scanning Keys by Type

```python
import redis

client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def get_keys_by_type(pattern='*', target_type='hash'):
    """Return all keys of a specific type matching a pattern."""
    matching = []
    cursor = 0

    while True:
        cursor, keys = client.scan(cursor, match=pattern, count=100)
        for key in keys:
            if client.type(key) == target_type:
                matching.append(key)
        if cursor == 0:
            break

    return matching

hash_keys = get_keys_by_type(pattern='user:*', target_type='hash')
print(f"Hash keys matching 'user:*': {hash_keys}")
```

## Summary

`TYPE` identifies the data structure stored at a Redis key, returning `string`, `list`, `set`, `zset`, `hash`, `stream`, or `none`. Use it before operating on keys to prevent `WRONGTYPE` errors, build type-aware key inspection tools, and classify keys when scanning large keyspaces. It is an O(1) operation and safe to use frequently.
