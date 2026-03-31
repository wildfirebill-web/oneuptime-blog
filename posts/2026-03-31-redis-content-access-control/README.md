# How to Implement Content Access Control with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Access Control, Authorization, Content, Backend

Description: Implement fast content access control with Redis sets and hashes to enforce per-user and role-based permissions on content items with sub-millisecond checks.

---

Checking whether a user can read or edit a piece of content on every request needs to be fast. Hitting a relational database for permission lookups adds latency under load. Redis sets and hashes let you cache and evaluate permissions in microseconds.

## Permission Model

Each content item has an access control list (ACL) stored as a Redis hash. Roles are stored as sets for fast membership checks:

```bash
# Grant user 42 read+write on article-101
HSET acl:article-101 user:42 "read,write"

# Grant all editors read+write
HSET acl:article-101 role:editor "read,write"

# Grant everyone read access
HSET acl:article-101 role:public "read"
```

## Checking Permission

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def has_permission(user_id: int, content_id: str, action: str) -> bool:
    acl_key = f"acl:{content_id}"

    # Check direct user permission
    user_perms = r.hget(acl_key, f"user:{user_id}") or ""
    if action in user_perms.split(","):
        return True

    # Check role-based permissions
    user_roles = r.smembers(f"user:{user_id}:roles")
    for role in user_roles:
        role_perms = r.hget(acl_key, f"role:{role}") or ""
        if action in role_perms.split(","):
            return True

    # Check public access
    public_perms = r.hget(acl_key, "role:public") or ""
    return action in public_perms.split(",")
```

## Managing User Roles

```python
def assign_role(user_id: int, role: str):
    r.sadd(f"user:{user_id}:roles", role)

def revoke_role(user_id: int, role: str):
    r.srem(f"user:{user_id}:roles", role)

def get_user_roles(user_id: int) -> set:
    return r.smembers(f"user:{user_id}:roles")
```

## Granting and Revoking Access

```python
def grant_access(content_id: str, principal: str, permissions: list):
    """principal can be 'user:42' or 'role:editor'"""
    r.hset(f"acl:{content_id}", principal, ",".join(permissions))

def revoke_access(content_id: str, principal: str):
    r.hdel(f"acl:{content_id}", principal)
```

## Caching Permission Decisions

For hot paths, cache the permission decision with a short TTL:

```python
def has_permission_cached(user_id: int, content_id: str, action: str) -> bool:
    cache_key = f"perm_cache:{user_id}:{content_id}:{action}"
    cached = r.get(cache_key)
    if cached is not None:
        return cached == "1"
    result = has_permission(user_id, content_id, action)
    r.setex(cache_key, 60, "1" if result else "0")
    return result

def invalidate_permission_cache(user_id: int, content_id: str):
    for action in ("read", "write", "delete", "publish"):
        r.delete(f"perm_cache:{user_id}:{content_id}:{action}")
```

## Summary

Redis hashes store per-content ACLs while sets track user-role memberships. Checking both direct and role-based permissions takes two to three Redis lookups. A short-TTL permission cache reduces lookups to one for hot content. Cache invalidation on ACL changes keeps permissions consistent within seconds.

