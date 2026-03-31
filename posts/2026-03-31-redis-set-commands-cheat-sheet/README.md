# Redis Set Commands Cheat Sheet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Set, Command, Cheat Sheet, Unique

Description: Complete Redis set commands reference covering SADD, SMEMBERS, SISMEMBER, SUNION, SINTER, SDIFF, and random sampling operations.

---

Redis sets are unordered collections of unique strings. They support O(1) membership testing and powerful set operations (union, intersection, difference). Here is the complete command reference.

## Add and Remove Members

```bash
# Add one or more members
SADD tags "redis" "cache" "nosql"

# Remove one or more members
SREM tags "nosql"

# Check if member exists
SISMEMBER tags "redis"         # returns 1 or 0

# Check multiple members at once
SMISMEMBER tags "redis" "mysql" "nosql"

# Get number of members
SCARD tags
```

## Reading Members

```bash
# Get all members (not recommended for large sets)
SMEMBERS tags

# Get N random members (no removal)
SRANDMEMBER tags 3     # 3 unique random members
SRANDMEMBER tags -3    # 3 random members (may repeat)

# Pop N random members (removes them)
SPOP tags 2
```

## Set Operations

```bash
# Union (all members from all sets)
SUNION set1 set2 set3
SUNIONSTORE dest set1 set2    # store result in dest

# Intersection (members in ALL sets)
SINTER set1 set2
SINTERSTORE dest set1 set2

# Difference (members in first set but NOT in others)
SDIFF set1 set2
SDIFFSTORE dest set1 set2

# Intersection cardinality (Redis 7.0+, no full computation needed)
SINTERCARD 2 set1 set2
SINTERCARD 2 set1 set2 LIMIT 10   # stop counting at 10
```

## Iterating Large Sets

```bash
# Cursor-based scan (safe for large sets)
SSCAN myset 0 COUNT 100
SSCAN myset 0 MATCH "user:*"
```

## Move Members Between Sets

```bash
# Atomically move member from one set to another
SMOVE source destination "member"
```

## Common Patterns

```bash
# Track unique visitors per page (deduplicated)
SADD visitors:page:/home "user:42" "user:99"
SCARD visitors:page:/home   # unique visitor count

# Tags system
SADD post:1:tags "redis" "tutorial" "beginner"
SADD post:2:tags "redis" "advanced"

# Find posts with common tags
SINTER post:1:tags post:2:tags   # returns "redis"

# Online users set
SADD online_users "user:42"
SREM online_users "user:42"
SISMEMBER online_users "user:42"

# Mutual friends
SINTER friends:alice friends:bob  # friends in common

# Permissions
SADD role:admin "read" "write" "delete" "admin"
SISMEMBER role:admin "write"
```

## Summary

Redis set commands enable unique value storage, fast membership checks, and set algebra operations like union, intersection, and difference. SINTERCARD avoids loading full sets when you only need the count, and SSCAN safely iterates large sets without blocking the server.
