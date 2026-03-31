# How to Use BF.EXISTS and BF.MEXISTS in Redis to Check Bloom Filters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Bloom Filter, RedisBloom, Command, NoSQL

Description: Learn how to use BF.EXISTS and BF.MEXISTS in Redis to check whether one or multiple items are present in a Bloom filter with probabilistic accuracy.

---

## Overview

After inserting items into a Redis Bloom filter, you use `BF.EXISTS` and `BF.MEXISTS` to test membership. `BF.EXISTS` checks a single item, while `BF.MEXISTS` checks multiple items in one call. Both commands return results immediately and are O(k) per item checked. Bloom filters can produce false positives but never false negatives - if the result is `0`, the item is definitely not in the filter.

## Prerequisites

Redis Stack or RedisBloom module must be running:

```bash
docker run -p 6379:6379 redis/redis-stack:latest
```

## Setting Up a Bloom Filter

```bash
BF.RESERVE myfilter 0.001 100000
BF.MADD myfilter "apple" "banana" "cherry" "date"
```

## Using BF.EXISTS

Check whether a single item exists in the filter:

```bash
BF.EXISTS myfilter "apple"
# Returns 1 (probably exists)

BF.EXISTS myfilter "mango"
# Returns 0 (definitely does not exist)
```

Return values:
- `1` - item was probably inserted (may be a false positive)
- `0` - item was definitely never inserted

## Using BF.MEXISTS

Check multiple items at once and get an array of results:

```bash
BF.MEXISTS myfilter "apple" "mango" "banana" "kiwi"
```

Output:

```text
1) (integer) 1
2) (integer) 0
3) (integer) 1
4) (integer) 0
```

## Understanding False Positives

Bloom filters trade exactness for space efficiency. A false positive occurs when the filter returns `1` for an item that was never inserted. The false positive rate is controlled by `BF.RESERVE`:

```bash
BF.RESERVE strict_filter 0.0001 1000000
# 0.01% false positive rate, capacity 1 million
```

A false negative is impossible - if `BF.EXISTS` returns `0`, the item was never added.

## Using BF.EXISTS in Python

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

r.execute_command('BF.RESERVE', 'userids', 0.001, 500000)
r.execute_command('BF.MADD', 'userids', 'uid-100', 'uid-200', 'uid-300')

# Single check
exists = r.execute_command('BF.EXISTS', 'userids', 'uid-100')
print(f"uid-100 exists: {bool(exists)}")  # True

# Multi check
results = r.execute_command('BF.MEXISTS', 'userids', 'uid-100', 'uid-999', 'uid-200')
for uid, res in zip(['uid-100', 'uid-999', 'uid-200'], results):
    print(f"{uid}: {'possibly exists' if res else 'definitely absent'}")
```

## Using BF.EXISTS in Node.js

```javascript
const { createClient } = require('redis');

async function checkBloomFilter() {
  const client = createClient();
  await client.connect();

  await client.sendCommand(['BF.RESERVE', 'emails', '0.001', '100000']);
  await client.sendCommand(['BF.MADD', 'emails', 'alice@example.com', 'bob@example.com']);

  const single = await client.sendCommand(['BF.EXISTS', 'emails', 'alice@example.com']);
  console.log('alice exists:', single === 1); // true

  const multi = await client.sendCommand([
    'BF.MEXISTS', 'emails',
    'alice@example.com',
    'carol@example.com',
    'bob@example.com'
  ]);
  console.log('Multi results:', multi); // [1, 0, 1]

  await client.disconnect();
}

checkBloomFilter();
```

## Practical Use Case - Cache Pre-check

Before querying a slow database, use `BF.EXISTS` to quickly rule out items that were never stored:

```python
def get_user(user_id):
    # Fast bloom filter check
    might_exist = r.execute_command('BF.EXISTS', 'known_users', user_id)
    if not might_exist:
        return None  # Definitely not in DB, skip query

    # May exist - query database
    return db.query("SELECT * FROM users WHERE id = ?", user_id)
```

For batch lookups, use `BF.MEXISTS`:

```python
def filter_existing_users(user_ids):
    results = r.execute_command('BF.MEXISTS', 'known_users', *user_ids)
    candidates = [uid for uid, exists in zip(user_ids, results) if exists]
    return candidates  # Only IDs that might be in DB
```

## BF.EXISTS vs BF.MEXISTS Comparison

| Command | Items Checked | Return Type | Use Case |
|---------|--------------|-------------|----------|
| BF.EXISTS | 1 | Integer | Single lookup |
| BF.MEXISTS | N | Array | Batch lookup |

## Summary

`BF.EXISTS` and `BF.MEXISTS` let you test Bloom filter membership for one or many items respectively. A result of `0` guarantees absence; a result of `1` means the item was probably inserted. Use these commands to quickly eliminate negative cases before querying slower data stores, dramatically reducing unnecessary database load.
