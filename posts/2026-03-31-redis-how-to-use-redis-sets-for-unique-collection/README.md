# How to Use Redis Sets for Unique Collections (Beginner Guide)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Sets, Unique Collections, Beginner, Node.js, Python, Membership

Description: A beginner-friendly guide to Redis Sets for storing unique items, checking membership, and performing set operations like union, intersection, and difference.

---

## What Are Redis Sets?

A Redis Set is an unordered collection of unique strings. Duplicate values are automatically ignored. Sets support O(1) add, remove, and membership check, plus set math operations (union, intersection, difference).

```bash
# Add members to a set
SADD fruits "apple" "banana" "orange"

# Try adding a duplicate - it's silently ignored
SADD fruits "apple"  # Still only 3 members

# Check if a member exists
SISMEMBER fruits "banana"
# 1 (yes)

SISMEMBER fruits "grape"
# 0 (no)

# Get all members
SMEMBERS fruits
# 1) "orange"
# 2) "banana"
# 3) "apple"  (order not guaranteed)
```

## Basic Set Commands

```bash
# Add members
SADD myset "a" "b" "c"

# Remove a member
SREM myset "b"

# Check membership
SISMEMBER myset "a"   # 1 = yes, 0 = no

# Count members
SCARD myset

# Get all members
SMEMBERS myset

# Get a random member
SRANDMEMBER myset

# Pop (remove and return) a random member
SPOP myset

# Check if multiple members exist (Redis 6.2+)
SMISMEMBER myset "a" "b" "z"
# 1) 1 (a exists)
# 2) 0 (b was removed)
# 3) 0 (z doesn't exist)
```

## Use Case 1: Tags

```javascript
const Redis = require('ioredis');
const redis = new Redis({ host: 'localhost', port: 6379 });

async function addTagsToPost(postId, tags) {
  await redis.sadd(`post:${postId}:tags`, ...tags);
}

async function getPostTags(postId) {
  return redis.smembers(`post:${postId}:tags`);
}

async function removeTag(postId, tag) {
  await redis.srem(`post:${postId}:tags`, tag);
}

async function hasTag(postId, tag) {
  return redis.sismember(`post:${postId}:tags`, tag);
}

// Usage
await addTagsToPost(101, ['redis', 'database', 'tutorial']);
const tags = await getPostTags(101);
console.log('Tags:', tags); // ['redis', 'database', 'tutorial']
```

## Use Case 2: Online Users

```javascript
async function setUserOnline(userId) {
  await redis.sadd('users:online', userId);
}

async function setUserOffline(userId) {
  await redis.srem('users:online', userId);
}

async function isUserOnline(userId) {
  const result = await redis.sismember('users:online', userId);
  return result === 1;
}

async function getOnlineCount() {
  return redis.scard('users:online');
}

async function getOnlineUsers() {
  return redis.smembers('users:online');
}
```

## Use Case 3: Unique Visitor Tracking

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def track_visitor(page: str, user_id: str):
    today = time.strftime('%Y-%m-%d')
    key = f"visitors:{page}:{today}"
    r.sadd(key, user_id)
    r.expire(key, 7 * 86400)  # Keep for 7 days

def get_unique_visitor_count(page: str, date: str) -> int:
    key = f"visitors:{page}:{date}"
    return r.scard(key)

# Track visits
track_visitor('homepage', 'user-42')
track_visitor('homepage', 'user-15')
track_visitor('homepage', 'user-42')  # Duplicate - ignored

print(get_unique_visitor_count('homepage', '2026-03-31'))  # 2
```

## Set Operations

The real power of Sets is in combining them:

```bash
# Union: all members from both sets
SUNION set1 set2
SUNIONSTORE destination set1 set2  # Store result

# Intersection: members in ALL sets
SINTER set1 set2 set3
SINTERSTORE destination set1 set2 set3

# Difference: members in set1 but NOT in set2
SDIFF set1 set2
SDIFFSTORE destination set1 set2
```

## Set Operations in Practice

```javascript
// Find common friends
async function getMutualFriends(userId1, userId2) {
  const result = await redis.sinter(
    `user:${userId1}:friends`,
    `user:${userId2}:friends`
  );
  return result;
}

// Find all tags across multiple posts
async function getAllTagsForPosts(postIds) {
  const keys = postIds.map(id => `post:${id}:tags`);
  return redis.sunion(...keys);
}

// Find users who liked post1 but NOT post2
async function usersWhoLikedPost1NotPost2(post1Id, post2Id) {
  return redis.sdiff(`post:${post1Id}:likes`, `post:${post2Id}:likes`);
}
```

```python
# Users interested in redis AND python (intersection)
r.sadd('interest:redis', 'user-1', 'user-2', 'user-3')
r.sadd('interest:python', 'user-2', 'user-3', 'user-4')

common = r.sinter('interest:redis', 'interest:python')
print(common)  # {'user-2', 'user-3'}

# Users interested in redis OR python (union)
all_interested = r.sunion('interest:redis', 'interest:python')
print(all_interested)  # {'user-1', 'user-2', 'user-3', 'user-4'}
```

## Use Case 4: Simple Rate Limiting with Sets

```python
def is_rate_limited(user_id: str, window_seconds: int = 60, max_requests: int = 10) -> bool:
    """Allow max N unique actions per window per user."""
    key = f"actions:{user_id}:{int(time.time() // window_seconds)}"
    r.sadd(key, str(time.time()))  # Add unique timestamp
    r.expire(key, window_seconds)
    count = r.scard(key)
    return count > max_requests
```

## Checking Multiple Memberships Efficiently

```javascript
// Check if user has all required roles
async function hasAllRoles(userId, requiredRoles) {
  const results = await redis.smismember(`user:${userId}:roles`, ...requiredRoles);
  return results.every(r => r === 1);
}

// Check if user has any of the specified permissions
async function hasAnyPermission(userId, permissions) {
  const userPerms = await redis.smembers(`user:${userId}:permissions`);
  return permissions.some(p => userPerms.includes(p));
}
```

## Summary

Redis Sets excel at storing unique items and performing membership checks in O(1) time. They are ideal for tags, online user tracking, unique visitor counting, and friend relationships. Set operations (union, intersection, difference) enable powerful queries like "mutual friends" or "all users interested in both topics" in a single Redis command. When uniqueness matters and order does not, Sets are the right data structure.
