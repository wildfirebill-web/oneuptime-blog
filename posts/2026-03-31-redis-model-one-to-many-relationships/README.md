# How to Model One-to-Many Relationships in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Data Modeling, Set, List

Description: Model one-to-many relationships in Redis using sets, lists, and sorted sets to represent parent-child collections with efficient lookup and traversal.

---

One-to-many relationships are common in almost every application: one user has many orders, one post has many comments, one team has many members. Redis provides several structures for these relationships.

## Approach 1: Set for Unordered Children

Use a set to track all child IDs belonging to a parent:

```bash
# User 42 follows products 5, 9, 17
SADD user:42:watchlist 5 9 17

# Add new product to watchlist
SADD user:42:watchlist 23

# Check membership
SISMEMBER user:42:watchlist 9   # 1 (yes)

# Get all items
SMEMBERS user:42:watchlist  # {5, 9, 17, 23}

# Remove
SREM user:42:watchlist 9
```

## Approach 2: Sorted Set for Ordered Children

Use a sorted set when insertion order or ranking matters:

```bash
# Team 7's members, sorted by join timestamp
ZADD team:7:members 1711900000 "user:1"
ZADD team:7:members 1711900100 "user:42"
ZADD team:7:members 1711900200 "user:99"

# Get members in join order
ZRANGE team:7:members 0 -1  # oldest to newest
ZREVRANGE team:7:members 0 -1  # newest first

# Get members who joined after a timestamp
ZRANGEBYSCORE team:7:members 1711900050 +inf
```

## Approach 3: List for Queue-Style Children

Use a list when order matters and you need head/tail access:

```bash
# Recent comments on post 5 (newest first)
LPUSH post:5:comments "comment:301" "comment:302"
LRANGE post:5:comments 0 9  # first 10 comments

# Cap list at 1000 comments
LTRIM post:5:comments 0 999
```

## Full Example: User Orders

```python
import redis
import time

r = redis.Redis(decode_responses=True)

def create_order(r, user_id, order_id, total):
    """Create an order and link it to the user."""
    pipe = r.pipeline()
    pipe.hset(f"order:{order_id}", mapping={
        "user_id": user_id,
        "total": total,
        "status": "pending",
        "created": str(int(time.time()))
    })
    # Add to user's order set (sorted by creation time)
    pipe.zadd(f"user:{user_id}:orders", {str(order_id): time.time()})
    pipe.execute()

def get_user_orders(r, user_id, page=0, per_page=10):
    """Paginate through a user's orders, newest first."""
    start = page * per_page
    stop = start + per_page - 1
    order_ids = r.zrevrange(f"user:{user_id}:orders", start, stop)
    pipe = r.pipeline(transaction=False)
    for oid in order_ids:
        pipe.hgetall(f"order:{oid}")
    return pipe.execute()

def delete_order(r, user_id, order_id):
    """Delete an order and remove it from the user's set."""
    pipe = r.pipeline()
    pipe.delete(f"order:{order_id}")
    pipe.zrem(f"user:{user_id}:orders", str(order_id))
    pipe.execute()

create_order(r, user_id=42, order_id=1001, total=49.99)
create_order(r, user_id=42, order_id=1002, total=12.50)

orders = get_user_orders(r, user_id=42)
for order in orders:
    print(order)
```

## Getting the Count

```bash
# Number of orders for user 42
ZCARD user:42:orders

# Number of members in a set
SCARD user:42:watchlist
```

## Reverse Lookup: Child to Parent

If you need to look up the parent from a child:

```bash
# Store parent reference directly in the child hash
HSET order:1001 user_id 42 total 49.99

# Or store separately
SET order:1001:owner 42
```

## Handling Deletion Consistently

When deleting a parent, clean up all children:

```python
def delete_user(r, user_id):
    order_ids = r.zrange(f"user:{user_id}:orders", 0, -1)
    pipe = r.pipeline()
    for oid in order_ids:
        pipe.delete(f"order:{oid}")
    pipe.delete(f"user:{user_id}:orders")
    pipe.delete(f"user:{user_id}")
    pipe.execute()
```

## Choosing the Right Structure

```text
Use case                            Structure
Unordered membership (tags, roles)  Set (SADD/SISMEMBER)
Ordered by score/time               Sorted Set (ZADD/ZRANGE)
Queue with head/tail access         List (LPUSH/RPOP)
Fixed-size recent history           List + LTRIM
Count of children important         Any (SCARD/ZCARD/LLEN)
```

## Summary

Model Redis one-to-many relationships with sets for unordered membership, sorted sets for time or score ordering, and lists for queue-style access. Store child IDs in the collection key and child data in separate hash keys. Always update the collection and the child atomically with pipelining, and implement reverse lookup (child to parent) by embedding the parent ID in the child hash.
