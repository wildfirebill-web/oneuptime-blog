# How to Use SMISMEMBER in Redis to Check Multiple Members

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Sets, SMISMEMBER, Membership, Commands

Description: Learn how to use SMISMEMBER in Redis to check membership of multiple elements in a set with a single command, reducing round trips.

---

## What Is SMISMEMBER

`SMISMEMBER` checks whether multiple elements are members of a Redis set in a single command. It returns an array of `1` or `0` values corresponding to each queried element - `1` if the element is in the set, `0` if it is not.

It was introduced in Redis 6.2 as a batch version of `SISMEMBER`, eliminating the need to send multiple individual membership checks.

## Syntax

```text
SMISMEMBER key member [member ...]
```

- `key` - the set key to check against
- `member [member ...]` - one or more elements to check

Returns an array of integers - `1` for each member present, `0` for each member absent. The order matches the order of the queried members.

## Basic Usage

### Check Multiple Members

```bash
redis-cli SADD fruits "apple" "banana" "cherry" "date"

redis-cli SMISMEMBER fruits "apple" "grape" "banana" "mango"
```

```text
1) (integer) 1
2) (integer) 0
3) (integer) 1
4) (integer) 0
```

- `apple` - present (1)
- `grape` - absent (0)
- `banana` - present (1)
- `mango` - absent (0)

### Check a Single Member

`SMISMEMBER` works with a single member too, though `SISMEMBER` is the conventional choice:

```bash
redis-cli SMISMEMBER fruits "cherry"
```

```text
1) (integer) 1
```

### All Members Absent

```bash
redis-cli SMISMEMBER fruits "kiwi" "papaya"
```

```text
1) (integer) 0
2) (integer) 0
```

## SMISMEMBER vs SISMEMBER

`SISMEMBER` checks one element at a time. For multiple checks, `SMISMEMBER` is more efficient:

```bash
# Old way - 3 round trips
redis-cli SISMEMBER fruits "apple"
redis-cli SISMEMBER fruits "banana"
redis-cli SISMEMBER fruits "cherry"

# New way - 1 round trip
redis-cli SMISMEMBER fruits "apple" "banana" "cherry"
```

## Practical Examples

### Batch Permission Check

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Allowed operations for role
r.sadd('role:admin:permissions', 'read', 'write', 'delete', 'manage_users')
r.sadd('role:editor:permissions', 'read', 'write')

def check_permissions(role, operations):
    """Check if a role has all required permissions."""
    key = f'role:{role}:permissions'
    results = r.smismember(key, *operations)
    return dict(zip(operations, [bool(r) for r in results]))

admin_perms = check_permissions('admin', ['read', 'delete', 'export'])
print(admin_perms)
# {'read': True, 'delete': True, 'export': False}

editor_perms = check_permissions('editor', ['read', 'delete'])
print(editor_perms)
# {'read': True, 'delete': False}
```

### Feature Flag Check for Multiple Features

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Enabled features
r.sadd('features:enabled', 'dark_mode', 'beta_dashboard', 'new_checkout')

def get_feature_flags(features):
    """Return dict of feature name to enabled status."""
    results = r.smismember('features:enabled', *features)
    return {feature: bool(enabled) for feature, enabled in zip(features, results)}

flags = get_feature_flags(['dark_mode', 'new_checkout', 'ai_assistant', 'beta_dashboard'])
print(flags)
# {'dark_mode': True, 'new_checkout': True, 'ai_assistant': False, 'beta_dashboard': True}

# Check if all required features are enabled
required = ['dark_mode', 'new_checkout']
all_enabled = all(flags[f] for f in required)
print(f"All required features enabled: {all_enabled}")
```

### Blocklist Filtering

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Blocked user IDs
r.sadd('blocklist:users', 'user:123', 'user:456', 'user:789')

def filter_blocked_users(user_ids):
    """Return only non-blocked users."""
    results = r.smismember('blocklist:users', *user_ids)
    return [uid for uid, blocked in zip(user_ids, results) if not blocked]

candidates = ['user:100', 'user:123', 'user:200', 'user:456', 'user:300']
allowed = filter_blocked_users(candidates)
print(f"Allowed users: {allowed}")
# ['user:100', 'user:200', 'user:300']
```

### Tag Validation

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Valid taxonomy tags
r.sadd('taxonomy:tags', 'python', 'javascript', 'redis', 'database', 'backend', 'frontend')

def validate_tags(tags):
    """Return valid and invalid tags from a list."""
    results = r.smismember('taxonomy:tags', *tags)
    valid = [t for t, r in zip(tags, results) if r]
    invalid = [t for t, r in zip(tags, results) if not r]
    return valid, invalid

user_tags = ['python', 'redis', 'nosql', 'cache', 'backend']
valid, invalid = validate_tags(user_tags)
print(f"Valid: {valid}")    # ['python', 'redis', 'backend']
print(f"Invalid: {invalid}") # ['nosql', 'cache']
```

## Summary

`SMISMEMBER` enables batch membership testing against a Redis set in a single round trip, returning an ordered array of `1`/`0` results matching the input order. It is ideal for permission checks, feature flag lookups, blocklist filtering, and tag validation where multiple elements need to be tested simultaneously. Always prefer it over multiple `SISMEMBER` calls when checking more than one element.
