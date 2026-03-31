# How to Design Redis Schemas for Evolving Requirements

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Schema, Design, Migration, Architecture

Description: Design Redis data schemas that can evolve safely over time using versioning, lazy migrations, and additive change strategies that avoid downtime or full data rewrites.

---

Redis has no schema enforcement - this is both a blessing and a curse. You can change your data model instantly, but without discipline you end up with inconsistent data across key versions. A versioned schema strategy and lazy migration approach let you evolve your Redis data models safely.

## Schema Versioning

Embed a version number in every stored object:

```python
import redis
import json

r = redis.Redis()
SCHEMA_VERSION = 2

def save_user(user_id, name, email, preferences=None):
    payload = {
        "_v": SCHEMA_VERSION,
        "name": name,
        "email": email,
        "preferences": preferences or {}
    }
    r.set(f"user:{user_id}", json.dumps(payload), ex=3600)
```

## Lazy Migration on Read

Migrate old schema versions automatically when they are read:

```python
def load_user(user_id):
    raw = r.get(f"user:{user_id}")
    if not raw:
        return None
    data = json.loads(raw)
    version = data.get("_v", 1)
    if version < SCHEMA_VERSION:
        data = migrate_user(data, from_version=version)
        # Write back the migrated version
        r.set(f"user:{user_id}", json.dumps(data), ex=3600)
    return data

def migrate_user(data, from_version):
    if from_version < 2:
        # v1 -> v2: add preferences field
        data["preferences"] = {}
        data["_v"] = 2
    return data
```

## Additive Changes

The safest schema changes are additive - add new fields, never remove or rename existing ones while old data exists:

```python
# v1 schema
{"_v": 1, "name": "Alice", "email": "alice@example.com"}

# v2 schema (additive - safe)
{"_v": 2, "name": "Alice", "email": "alice@example.com", "preferences": {}}

# v3 schema (additive - safe)
{"_v": 3, "name": "Alice", "email": "alice@example.com",
 "preferences": {}, "created_at": 1700000000}
```

Rename fields only after all old data has been migrated or expired.

## Hash Field Evolution

For hash-stored entities, adding fields is safe; removing requires a multi-phase approach:

```bash
# Add a new field - safe at any time
HSET user:1001 loyalty_tier gold

# Do not delete old fields until all code stops reading them
# Phase 1: Stop writing old field
# Phase 2: Stop reading old field
# Phase 3: Delete old field
HDEL user:1001 deprecated_field
```

## Background Migration Job

For urgent migrations, run a background job to update all existing keys:

```python
def background_migrate_users():
    cursor = 0
    migrated = 0
    while True:
        cursor, keys = r.scan(cursor, match="user:*", count=200)
        for key in keys:
            raw = r.get(key)
            if not raw:
                continue
            data = json.loads(raw)
            if data.get("_v", 1) < SCHEMA_VERSION:
                data = migrate_user(data, data.get("_v", 1))
                ttl = r.ttl(key)
                r.set(key, json.dumps(data), ex=ttl if ttl > 0 else 3600)
                migrated += 1
        if cursor == 0:
            break
    print(f"Migrated {migrated} users")
```

## Namespace Versioning for Breaking Changes

For truly breaking changes, use a new key namespace:

```bash
# Old schema
user:1001  -> v1 data

# New schema - new prefix
user_v2:1001 -> v2 data
```

Run both in parallel, migrate traffic, then drop the old namespace.

## Summary

Redis schema evolution is manageable through embedded version numbers, lazy read-time migration, and a strict additive-first change policy. Background migration jobs handle bulk updates for urgent changes, while namespace versioning provides an escape hatch for breaking changes that cannot be expressed as additive additions to existing data structures.
