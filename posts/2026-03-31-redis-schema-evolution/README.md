# How to Handle Schema Evolution in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Schema Evolution, Data Modeling, Migration, Hash

Description: Learn strategies for evolving Redis data schemas without downtime, including field versioning, lazy migration, and batch migration patterns.

---

Unlike relational databases, Redis has no schema enforcement - but that means schema changes can silently break your application if you are not careful. When you rename a field, add required data, or change a value format, old keys in Redis still hold the old shape. Here are the main strategies for handling this safely.

## Strategy 1: Version Field in Hash

Add a `_v` field to every Hash to track the schema version:

```bash
HSET user:1001 _v 1 name "Alice" email "alice@example.com"
```

When you read a key, check `_v` and upgrade if necessary:

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def get_user(user_id: str) -> dict:
    key = f"user:{user_id}"
    data = r.hgetall(key)
    version = int(data.get("_v", 1))

    if version < 2:
        # Migrate: rename 'email' to 'email_address'
        email = data.pop("email", None)
        if email:
            r.hset(key, "email_address", email)
            r.hdel(key, "email")
        r.hset(key, "_v", 2)
        data["email_address"] = email
        data["_v"] = "2"

    return data
```

This is called **lazy migration** - data is upgraded only when accessed.

## Strategy 2: Key Namespace Versioning

For larger schema changes, use a new key namespace and migrate in the background:

```bash
# Old schema
user:v1:1001

# New schema
user:v2:1001
```

Write to both namespaces during the transition, then migrate old keys to the new namespace and switch reads over:

```python
def write_user(user_id: str, data: dict):
    # Dual-write during migration
    r.hset(f"user:v1:{user_id}", mapping=data)
    r.hset(f"user:v2:{user_id}", mapping=transform_v2(data))
```

## Strategy 3: Batch Migration Script

After deploying the new code, migrate old keys in batches using SCAN:

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def migrate_users():
    cursor = 0
    migrated = 0
    while True:
        cursor, keys = r.scan(cursor, match="user:*", count=100)
        pipe = r.pipeline(transaction=False)
        for key in keys:
            data = r.hgetall(key)
            if data.get("_v") == "1":
                email = data.get("email")
                if email:
                    pipe.hset(key, "email_address", email)
                    pipe.hdel(key, "email")
                pipe.hset(key, "_v", 2)
                migrated += 1
        pipe.execute()
        if cursor == 0:
            break
    print(f"Migrated {migrated} keys")

migrate_users()
```

Use `count=100` with SCAN to avoid blocking Redis. Run this as a background job during off-peak hours.

## Strategy 4: Handling Removed Fields

When removing a field, first stop writing it, then delete it from existing keys during migration:

```bash
# Remove a field from all matching keys
SCAN 0 MATCH user:* COUNT 100
# For each key:
HDEL user:1001 deprecated_field
```

## Strategy 5: Default Values for New Fields

When adding an optional field, handle its absence in application code:

```python
def get_display_name(user_id: str) -> str:
    key = f"user:{user_id}"
    # New field may not exist in old records
    display_name = r.hget(key, "display_name")
    if display_name is None:
        name = r.hget(key, "name")
        return name or "Unknown"
    return display_name
```

## Summary

Redis schema evolution is handled in application code rather than by the database engine. Use a version field in your Hashes to enable lazy migration on read, key namespace versioning for large structural changes, and background SCAN-based batch migration to clean up old data. Always handle missing fields gracefully in your application to make new fields backward compatible from day one.
