# How to Use SREM in Redis to Remove Specific Set Members

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Set, SREM, Command, Data Structure

Description: Learn how to use SREM in Redis to remove one or more specific members from a set, with practical examples for managing dynamic collections.

---

## What Is SREM

`SREM` removes one or more specified members from a Redis set. If a member does not exist in the set, it is simply ignored. The command returns the count of members that were actually removed.

## Syntax

```text
SREM key member [member ...]
```

- `key` - the set key
- `member [member ...]` - one or more values to remove

Returns an integer - the number of members that were removed (not counting members that were not present).

## Basic Usage

### Remove a Single Member

```bash
redis-cli SADD tags "redis" "database" "cache" "nosql"

redis-cli SREM tags "cache"
```

```text
(integer) 1
```

### Remove Multiple Members

```bash
redis-cli SREM tags "nosql" "database"
```

```text
(integer) 2
```

### Remove Non-Existent Member

```bash
redis-cli SREM tags "mysql"
```

```text
(integer) 0
```

The command returns 0 but does not error.

### Remove Mix of Existing and Non-Existing

```bash
redis-cli SREM tags "redis" "nonexistent" "anothernonexistent"
```

```text
(integer) 1
```

Only `redis` was removed - count is 1.

### Verify After Removal

```bash
redis-cli SMEMBERS tags
```

```text
(empty array)
```

## Behavior When Key Does Not Exist

```bash
redis-cli SREM nonexistent_key "value"
```

```text
(integer) 0
```

No error is raised for non-existent keys.

## Practical Examples

### Session Management - Remove Expired Sessions

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Track active sessions
r.sadd('active_sessions', 'sess:abc', 'sess:def', 'sess:ghi', 'sess:jkl')

def logout_user(session_id):
    """Remove a session on logout."""
    removed = r.srem('active_sessions', session_id)
    if removed:
        print(f"Session {session_id} removed")
    else:
        print(f"Session {session_id} was not active")

logout_user('sess:abc')  # "Session sess:abc removed"
logout_user('sess:xyz')  # "Session sess:xyz was not active"
```

### Access Control - Revoke Permissions

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

r.sadd('user:admin:roles', 'editor', 'moderator', 'viewer', 'publisher')

def revoke_roles(user_key, *roles):
    """Revoke specified roles from a user."""
    removed = r.srem(user_key, *roles)
    print(f"Revoked {removed} role(s) from {user_key}")
    return removed

revoke_roles('user:admin:roles', 'moderator', 'publisher')
# Revoked 2 role(s) from user:admin:roles

remaining = r.smembers('user:admin:roles')
print(f"Remaining roles: {remaining}")  # {'editor', 'viewer'}
```

### Unsubscribe from Topics

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

r.sadd('user:alice:subscriptions', 'topic:sports', 'topic:tech', 'topic:music', 'topic:news')

def unsubscribe(user_id, *topics):
    key = f'user:{user_id}:subscriptions'
    removed = r.srem(key, *topics)
    return removed

removed_count = unsubscribe('alice', 'topic:sports', 'topic:music')
print(f"Unsubscribed from {removed_count} topics")

subs = r.smembers('user:alice:subscriptions')
print(f"Active subscriptions: {subs}")  # {'topic:tech', 'topic:news'}
```

### Removing Items from a Cart

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

r.sadd('cart:user:42', 'sku:1001', 'sku:1002', 'sku:1003')

def remove_from_cart(user_id, *skus):
    key = f'cart:user:{user_id}'
    removed = r.srem(key, *skus)
    print(f"Removed {removed} item(s) from cart")

remove_from_cart(42, 'sku:1002', 'sku:9999')
# Removed 1 item(s) from cart

cart = r.smembers('cart:user:42')
print(f"Cart: {cart}")  # {'sku:1001', 'sku:1003'}
```

### Bulk Cleanup with Pipeline

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Many users to remove from a group
r.sadd('group:beta_testers', *[f'user:{i}' for i in range(1, 101)])

# Batch removal using pipeline
users_to_remove = [f'user:{i}' for i in range(1, 51)]

pipe = r.pipeline()
# SREM can take multiple members, so just one command is enough
pipe.srem('group:beta_testers', *users_to_remove)
pipe.scard('group:beta_testers')
results = pipe.execute()

print(f"Removed: {results[0]}, Remaining: {results[1]}")
# Removed: 50, Remaining: 50
```

## Summary

`SREM` removes one or more specified members from a Redis set and returns the count of successfully removed members. Non-existent members and non-existent keys are handled gracefully without errors. It is the standard way to unsubscribe users from topics, revoke permissions, remove items from collections, and manage any dynamic set-based membership in Redis applications.
