# How to Use CF.DEL in Redis to Delete from Cuckoo Filters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cuckoo Filter, RedisBloom, Commands, NoSql

Description: Learn how to use CF.DEL in Redis to remove items from a Cuckoo filter, and understand the implications of deleting items that were never inserted.

---

## Overview

`CF.DEL` removes an item from a Cuckoo filter in Redis. This is one of the key advantages Cuckoo filters have over Bloom filters - deletion is supported. The command decrements the count for items inserted multiple times using `CF.ADD`. However, deleting an item that was never inserted can corrupt the filter by creating a false deletion, so caution is required.

## Prerequisites

Redis Stack or RedisBloom module:

```bash
docker run -p 6379:6379 redis/redis-stack:latest
```

## Setting Up and Populating a Filter

```bash
CF.RESERVE active_sessions 100000
CF.ADD active_sessions "sess-abc"
CF.ADD active_sessions "sess-def"
CF.ADD active_sessions "sess-ghi"
```

## Using CF.DEL

```bash
CF.DEL active_sessions "sess-abc"
# Returns 1 - item was found and deleted

CF.DEL active_sessions "sess-xyz"
# Returns 0 - item was not found
```

Return values:
- `1` - item was found and deleted
- `0` - item was not found in the filter

## Deleting Duplicate Items

When `CF.ADD` is used to insert the same item multiple times, `CF.DEL` removes one occurrence at a time:

```bash
CF.ADD myfilter "event-99"
CF.ADD myfilter "event-99"
CF.ADD myfilter "event-99"

CF.COUNT myfilter "event-99"
# Returns 3

CF.DEL myfilter "event-99"
CF.COUNT myfilter "event-99"
# Returns 2

CF.DEL myfilter "event-99"
CF.DEL myfilter "event-99"
CF.COUNT myfilter "event-99"
# Returns 0
```

## Warning: Deleting Uninserted Items

Deleting an item that was never inserted corrupts the filter because Cuckoo filters cannot distinguish between a real deletion and a phantom deletion:

```bash
# DANGER: deleting something never added
CF.DEL myfilter "ghost-item"
# This may delete a fingerprint belonging to a real item
```

Always track what you have inserted to avoid false deletions.

## Using CF.DEL in Python

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

r.execute_command('CF.RESERVE', 'active_tokens', 200000)
r.execute_command('CF.ADD', 'active_tokens', 'token-aaa')
r.execute_command('CF.ADD', 'active_tokens', 'token-bbb')

# Revoke a token
def revoke_token(token):
    result = r.execute_command('CF.DEL', 'active_tokens', token)
    if result == 1:
        print(f"Token {token} revoked")
    else:
        print(f"Token {token} was not active")

revoke_token('token-aaa')
revoke_token('token-xxx')  # Not found
```

## Using CF.DEL in Node.js

```javascript
const { createClient } = require('redis');

async function deleteFromCuckoo(filterName, item) {
  const client = createClient();
  await client.connect();

  const result = await client.sendCommand(['CF.DEL', filterName, item]);

  if (result === 1) {
    console.log(`"${item}" removed from "${filterName}"`);
  } else {
    console.log(`"${item}" was not found in "${filterName}"`);
  }

  await client.disconnect();
  return result;
}

deleteFromCuckoo('active_sessions', 'sess-def');
```

## Practical Use Case - Session Management

Cuckoo filters are ideal for session invalidation because you need both insertion and deletion:

```python
def create_session(session_id):
    r.execute_command('CF.ADDNX', 'valid_sessions', session_id)
    # Store full session data elsewhere
    r.setex(f'session:{session_id}', 3600, 'session_data')

def invalidate_session(session_id):
    r.execute_command('CF.DEL', 'valid_sessions', session_id)
    r.delete(f'session:{session_id}')

def is_session_valid(session_id):
    # Fast pre-check
    maybe_valid = r.execute_command('CF.EXISTS', 'valid_sessions', session_id)
    if not maybe_valid:
        return False
    # Confirm session data exists
    return r.exists(f'session:{session_id}')
```

## CF.DEL vs Other Deletion Strategies

| Strategy | Supports Delete | Space Efficient | Notes |
|----------|----------------|----------------|-------|
| CF.DEL | Yes | Yes | Cuckoo filter native |
| BF (Bloom) | No | Very yes | Cannot delete |
| Redis SET | Yes | No | Full set membership, more memory |

## Summary

`CF.DEL` removes one occurrence of an item from a Cuckoo filter and returns `1` on success or `0` if not found. For items inserted multiple times, multiple `CF.DEL` calls are needed to fully remove all occurrences. Never delete items that were not inserted, as this can corrupt the filter. Cuckoo filter deletion makes it ideal for session tracking, token revocation, and other use cases requiring both add and remove operations.
