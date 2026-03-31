# How to Use Redis Hashes to Store Objects (Beginner Guide)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Hashes, Data Structures, Beginner, Objects, Node.js, Python

Description: A beginner-friendly guide to storing structured objects in Redis Hashes, with practical examples for user profiles, product data, and configuration.

---

## What Are Redis Hashes?

A Redis Hash is a collection of field-value pairs stored under a single key. Think of it as a dictionary or a flat object. Hashes are ideal for storing structured data like user profiles, product details, or configuration settings.

```bash
# Create a hash with multiple fields at once
HSET user:42 name "Alice" email "alice@example.com" age 30 role "admin"

# Get a specific field
HGET user:42 name
# "Alice"

# Get all fields and values
HGETALL user:42
# 1) "name"
# 2) "Alice"
# 3) "email"
# 4) "alice@example.com"
# 5) "age"
# 6) "30"
# 7) "role"
# 8) "admin"
```

## Hash vs. String for Objects

```bash
# Option 1: Store entire object as JSON string
SET user:42:json '{"name":"Alice","email":"alice@example.com","age":30}'

# Option 2: Store as a Hash (individual fields)
HSET user:42 name "Alice" email "alice@example.com" age 30
```

**Use Hash when:**
- You frequently update individual fields
- You only need specific fields (HMGET is efficient)
- The object has a manageable number of fields (< 100)

**Use JSON string when:**
- You always read the entire object at once
- The object has deeply nested structure

## Basic Hash Commands

```bash
# Set a single field
HSET user:42 city "New York"

# Get a single field
HGET user:42 city

# Get multiple specific fields
HMGET user:42 name email
# 1) "Alice"
# 2) "alice@example.com"

# Check if a field exists
HEXISTS user:42 phone
# 0 (does not exist)

# Delete a field
HDEL user:42 age

# Count number of fields
HLEN user:42

# Get all field names
HKEYS user:42

# Get all values
HVALS user:42

# Set field only if it doesn't exist
HSETNX user:42 created_at 1711900000
```

## Node.js Examples

```javascript
const Redis = require('ioredis');
const redis = new Redis({ host: 'localhost', port: 6379 });

// Create a user profile
async function createUser(userId, userData) {
  const key = `user:${userId}`;

  await redis.hset(key, {
    name: userData.name,
    email: userData.email,
    role: userData.role || 'user',
    createdAt: Date.now().toString()
  });

  // Set TTL if needed
  await redis.expire(key, 86400); // 24 hours
}

// Get a specific field
async function getUserEmail(userId) {
  return redis.hget(`user:${userId}`, 'email');
}

// Get selected fields
async function getUserBasicInfo(userId) {
  const [name, email, role] = await redis.hmget(`user:${userId}`, 'name', 'email', 'role');
  return { userId, name, email, role };
}

// Get all fields
async function getFullProfile(userId) {
  return redis.hgetall(`user:${userId}`);
}

// Update a single field
async function updateUserEmail(userId, newEmail) {
  await redis.hset(`user:${userId}`, 'email', newEmail);
  await redis.hset(`user:${userId}`, 'updatedAt', Date.now().toString());
}

// Delete a field
async function removeUserField(userId, field) {
  await redis.hdel(`user:${userId}`, field);
}

// Usage
await createUser(42, { name: 'Alice', email: 'alice@example.com', role: 'admin' });
const email = await getUserEmail(42);
const profile = await getFullProfile(42);
console.log(profile);
```

## Python Examples

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def create_product(product_id: int, product: dict):
    key = f"product:{product_id}"
    r.hset(key, mapping={
        'name': product['name'],
        'price': str(product['price']),
        'stock': str(product['stock']),
        'category': product['category'],
        'created_at': str(int(time.time()))
    })

def get_product(product_id: int) -> dict | None:
    return r.hgetall(f"product:{product_id}") or None

def update_stock(product_id: int, quantity_change: int):
    """Atomically update stock count."""
    key = f"product:{product_id}"
    return r.hincrby(key, 'stock', quantity_change)

def get_product_price(product_id: int) -> float | None:
    price = r.hget(f"product:{product_id}", 'price')
    return float(price) if price else None

# Usage
create_product(101, {
    'name': 'Redis Book',
    'price': 29.99,
    'stock': 100,
    'category': 'books'
})

product = get_product(101)
print(product)

# Decrement stock when item is purchased
update_stock(101, -1)
```

## Numeric Field Operations

```bash
# Increment an integer field
HINCRBY user:42 login_count 1
# Useful for counters without needing to GET first

# Increment a float field
HINCRBYFLOAT product:101 price 5.00
# product price goes from 29.99 to 34.99
```

```javascript
// Atomic increment - no race conditions
async function recordLogin(userId) {
  const key = `user:${userId}`;
  await redis.hincrby(key, 'loginCount', 1);
  await redis.hset(key, 'lastLoginAt', Date.now().toString());
}
```

## Storing Configuration in Hashes

```bash
# Store app configuration
HSET config:app debug "false" version "2.1.0" max_connections "100"
HSET config:app feature_dark_mode "true" feature_beta "false"

# Read a config value
HGET config:app version

# Check feature flag
HGET config:app feature_dark_mode
```

## Memory Efficiency of Hashes

Redis Hashes are memory-efficient for small objects. Redis uses a compact encoding (listpack) when:
- The hash has fewer than 128 fields
- All values are smaller than 64 bytes

```bash
# Check the encoding of a hash
OBJECT ENCODING user:42
# "listpack" for small hashes
# "hashtable" for large hashes
```

## Summary

Redis Hashes are the natural data structure for storing structured objects in Redis. HSET stores fields, HGET retrieves individual fields efficiently without fetching the entire object, and HINCRBY atomically increments numeric fields. Use Hashes for user profiles, product details, and configuration where you frequently access specific fields. Keep hashes small (under 100 fields) to benefit from Redis's compact listpack encoding.
