# How to Test Redis Data Migration Scripts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Migration, Testing, Data Modeling, Script

Description: Learn how to safely test Redis data migration scripts using isolated test instances, fixture loading, and verification checks before running in production.

---

Redis migrations - renaming keys, changing value formats, restructuring data - carry real risk because Redis has no rollback. A migration script that silently drops data in production is hard to recover from. The safest approach is to build and test migration scripts the same way you would test application code.

## Set Up an Isolated Migration Test Environment

Never test migrations against production. Use a Docker container loaded with a snapshot of real data:

```bash
# Start an isolated test instance
docker run --name redis-migrate-test \
  -p 6381:6379 \
  -d redis:7

# Load a production RDB snapshot into it
docker cp dump.rdb redis-migrate-test:/data/dump.rdb
docker restart redis-migrate-test
```

## Write a Fixture Loader

If you do not have a real snapshot, load representative fixtures:

```python
import redis

r = redis.Redis(host="localhost", port=6381, decode_responses=True)

def load_fixtures():
    # Old schema: v1 user Hashes with 'email' field
    r.hset("user:1001", mapping={"_v": "1", "name": "Alice", "email": "alice@example.com"})
    r.hset("user:1002", mapping={"_v": "1", "name": "Bob", "email": "bob@example.com"})
    # Include edge cases
    r.hset("user:1003", mapping={"_v": "1", "name": "Charlie"})  # missing email
```

## Write the Migration Script

```python
def migrate_v1_to_v2(r: redis.Redis):
    cursor = 0
    migrated = 0
    skipped = 0
    while True:
        cursor, keys = r.scan(cursor, match="user:*", count=100)
        pipe = r.pipeline(transaction=False)
        for key in keys:
            data = r.hgetall(key)
            if data.get("_v") != "1":
                skipped += 1
                continue
            email = data.get("email")
            if email:
                pipe.hset(key, "email_address", email)
                pipe.hdel(key, "email")
            pipe.hset(key, "_v", "2")
            migrated += 1
        pipe.execute()
        if cursor == 0:
            break
    return {"migrated": migrated, "skipped": skipped}
```

## Write Verification Checks

After migration, verify correctness with automated assertions:

```python
def verify_migration(r: redis.Redis):
    errors = []
    cursor = 0
    while True:
        cursor, keys = r.scan(cursor, match="user:*", count=100)
        for key in keys:
            data = r.hgetall(key)
            if data.get("_v") != "2":
                errors.append(f"{key}: still at v1")
            if "email" in data:
                errors.append(f"{key}: old 'email' field still present")
        if cursor == 0:
            break
    return errors
```

## Test with pytest

```python
import pytest
import redis as redislib

@pytest.fixture
def r():
    client = redislib.Redis(host="localhost", port=6381, decode_responses=True)
    client.flushdb()
    load_fixtures()
    yield client
    client.flushdb()

def test_migration_updates_version(r):
    result = migrate_v1_to_v2(r)
    assert result["migrated"] == 3

def test_migration_renames_email_field(r):
    migrate_v1_to_v2(r)
    assert r.hget("user:1001", "email_address") == "alice@example.com"
    assert r.hget("user:1001", "email") is None

def test_migration_handles_missing_email(r):
    migrate_v1_to_v2(r)
    assert r.hget("user:1003", "_v") == "2"
    assert r.hget("user:1003", "email") is None

def test_verify_no_errors(r):
    migrate_v1_to_v2(r)
    errors = verify_migration(r)
    assert errors == []
```

## Dry-Run Mode

Add a `--dry-run` flag to your migration script so you can preview changes without applying them:

```python
def migrate_v1_to_v2(r: redis.Redis, dry_run: bool = False):
    cursor = 0
    changes = []
    while True:
        cursor, keys = r.scan(cursor, match="user:*", count=100)
        for key in keys:
            data = r.hgetall(key)
            if data.get("_v") == "1":
                changes.append(key)
                if not dry_run:
                    # apply changes
                    pass
        if cursor == 0:
            break
    return changes
```

## Summary

Test Redis migration scripts by loading fixtures into an isolated Docker instance, writing a verification function that checks post-migration state, and covering edge cases (missing fields, already-migrated keys) with pytest. Add a dry-run mode to preview scope before running in production, and always test against a copy of real data before touching live systems.
