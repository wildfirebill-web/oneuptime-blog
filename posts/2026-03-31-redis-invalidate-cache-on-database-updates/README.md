# How to Invalidate Redis Cache on Database Updates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cache Invalidation, Database, Consistency, Python, Node.js

Description: Implement Redis cache invalidation strategies including key-level delete, tag-based invalidation, and event-driven approaches to keep cache data consistent with database updates.

---

## Cache Invalidation Strategies

There are several strategies for invalidating Redis cache on database updates:

1. **Delete on write** - delete the cache key when the database record changes
2. **Tag-based invalidation** - group related keys by tag and delete all tagged keys
3. **Event-driven invalidation** - database events trigger cache invalidation asynchronously
4. **TTL-only** - let keys expire naturally (simplest, accepts eventual consistency)

## Strategy 1 - Delete on Write

```python
import redis
import psycopg2
import json

r = redis.Redis(decode_responses=True)
pg = psycopg2.connect(host="localhost", database="myapp", user="postgres", password="password")

def update_product(product_id: int, data: dict):
    with pg.cursor() as cur:
        set_clause = ", ".join(f"{k} = %s" for k in data.keys())
        cur.execute(f"UPDATE products SET {set_clause} WHERE id = %s", [*data.values(), product_id])
    pg.commit()

    # Invalidate specific product cache
    r.delete(f"product:{product_id}")
    print(f"Invalidated product:{product_id}")

def delete_product(product_id: int):
    with pg.cursor() as cur:
        cur.execute("DELETE FROM products WHERE id = %s", (product_id,))
    pg.commit()

    r.delete(f"product:{product_id}")

def create_product(data: dict) -> int:
    with pg.cursor() as cur:
        cols = ", ".join(data.keys())
        placeholders = ", ".join(["%s"] * len(data))
        cur.execute(f"INSERT INTO products ({cols}) VALUES ({placeholders}) RETURNING id", list(data.values()))
        product_id = cur.fetchone()[0]
    pg.commit()

    # Optionally write-through on create
    data["id"] = product_id
    r.set(f"product:{product_id}", json.dumps(data), ex=3600)

    # Invalidate list caches
    for key in r.scan_iter("products:list:*"):
        r.delete(key)

    return product_id
```

## Strategy 2 - Tag-Based Invalidation

Group cache keys by entity type or query pattern so all related caches can be invalidated with one operation:

```python
CACHE_TAG_TTL = 7200

def set_with_tags(cache_key: str, value, tags: list, ttl: int = 3600):
    pipe = r.pipeline()
    pipe.set(cache_key, json.dumps(value, default=str), ex=ttl)
    for tag in tags:
        tag_key = f"cache:tags:{tag}"
        pipe.sadd(tag_key, cache_key)
        pipe.expire(tag_key, CACHE_TAG_TTL)
    pipe.execute()

def invalidate_by_tag(tag: str):
    tag_key = f"cache:tags:{tag}"
    cache_keys = r.smembers(tag_key)
    if cache_keys:
        r.delete(*cache_keys)
        r.delete(tag_key)
        print(f"Invalidated {len(cache_keys)} keys for tag '{tag}'")

# Cache product with tags
def cache_product(product_id: int, product_data: dict):
    category = product_data.get("category", "unknown")
    set_with_tags(
        f"product:{product_id}",
        product_data,
        tags=[f"category:{category}", "products:all", f"product:{product_id}"],
        ttl=3600
    )

# When a product is updated
def update_product_with_tags(product_id: int, data: dict):
    with pg.cursor() as cur:
        cur.execute("SELECT category FROM products WHERE id = %s", (product_id,))
        old_category = cur.fetchone()[0]
        set_clause = ", ".join(f"{k} = %s" for k in data.keys())
        cur.execute(f"UPDATE products SET {set_clause} WHERE id = %s", [*data.values(), product_id])
    pg.commit()

    # Invalidate all caches tagged with this product
    invalidate_by_tag(f"product:{product_id}")
    # Invalidate category caches if category changed
    if "category" in data:
        invalidate_by_tag(f"category:{old_category}")
        invalidate_by_tag(f"category:{data['category']}")
```

## Strategy 3 - Event-Driven Invalidation via Pub/Sub

Publish invalidation events that all app instances subscribe to:

```python
def publish_invalidation_event(entity_type: str, entity_id, operation: str):
    event = json.dumps({
        "entity_type": entity_type,
        "entity_id": entity_id,
        "operation": operation
    })
    r.publish("cache:invalidation", event)

# Publisher side - called after DB update
def update_user(user_id: int, data: dict):
    with pg.cursor() as cur:
        set_clause = ", ".join(f"{k} = %s" for k in data.keys())
        cur.execute(f"UPDATE users SET {set_clause} WHERE id = %s", [*data.values(), user_id])
    pg.commit()
    publish_invalidation_event("user", user_id, "update")
```

```python
# Subscriber side - runs in a background thread on each app server
import threading

def start_invalidation_listener():
    sub = redis.Redis(decode_responses=True).pubsub()
    sub.subscribe("cache:invalidation")

    for message in sub.listen():
        if message["type"] != "message":
            continue
        event = json.loads(message["data"])
        entity_type = event["entity_type"]
        entity_id = event["entity_id"]
        op = event["operation"]

        key = f"{entity_type}:{entity_id}"
        r.delete(key)
        invalidate_by_tag(f"{entity_type}:{entity_id}")
        print(f"Invalidated {key} via event ({op})")

thread = threading.Thread(target=start_invalidation_listener, daemon=True)
thread.start()
```

## Node.js Example

```javascript
const redis = require("redis");
const { Pool } = require("pg");

const redisClient = redis.createClient();
const pg = new Pool();

async function updateUser(userId, updates) {
  const keys = Object.keys(updates);
  const values = Object.values(updates);
  const setClause = keys.map((k, i) => `${k} = $${i + 1}`).join(", ");

  await pg.query(`UPDATE users SET ${setClause} WHERE id = $${keys.length + 1}`, [...values, userId]);

  // Invalidate cache
  await redisClient.del(`user:${userId}`);
  console.log(`Invalidated cache for user:${userId}`);
}

async function deleteUser(userId) {
  await pg.query("DELETE FROM users WHERE id = $1", [userId]);
  await redisClient.del(`user:${userId}`);
}
```

## Atomic Update and Invalidate with Lua

For strict consistency, use a Lua script to atomically delete related keys:

```lua
local prefix = ARGV[1]
local cursor = "0"
repeat
    local result = redis.call("SCAN", cursor, "MATCH", prefix, "COUNT", 100)
    cursor = result[1]
    local keys = result[2]
    if #keys > 0 then
        redis.call("DEL", unpack(keys))
    end
until cursor == "0"
return 1
```

```python
invalidate_prefix = r.register_script("""
local prefix = ARGV[1]
local cursor = "0"
repeat
    local result = redis.call("SCAN", cursor, "MATCH", prefix, "COUNT", 100)
    cursor = result[1]
    local keys = result[2]
    if #keys > 0 then redis.call("DEL", unpack(keys)) end
until cursor == "0"
return 1
""")

invalidate_prefix(args=["users:list:*"])
```

## Summary

Cache invalidation on database updates uses delete-on-write for targeted key removal, tag-based invalidation for bulk removal of related query caches, and event-driven Pub/Sub for cross-process invalidation on distributed deployments. The most reliable approach combines direct key deletion for entity caches with tag-based invalidation for query caches, ensuring both specific and related entries are cleared when data changes.
