# How to Use TOPK.LIST in Redis to List Top-K Items

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Top-K, Heavy Hitters, Trending, Leaderboard

Description: Learn how to use TOPK.LIST in Redis to retrieve all items currently in the Top-K list, optionally with their estimated frequency counts.

---

## What Is TOPK.LIST?

`TOPK.LIST` returns all items currently tracked in a Redis Top-K structure. This gives you the full picture of the most frequent items observed, making it ideal for rendering trending lists, leaderboards, and popular content sections.

Unlike `TOPK.QUERY` which checks specific items, TOPK.LIST discovers what the top items are.

## Syntax

```text
TOPK.LIST key [WITHCOUNT]
```

- `WITHCOUNT` - optional flag to also return estimated counts for each item

## Basic Usage

```bash
# Setup
TOPK.RESERVE news:trending 5
TOPK.ADD news:trending "article:python" "article:redis" "article:docker"
TOPK.ADD news:trending "article:redis" "article:python" "article:redis"
TOPK.ADD news:trending "article:python" "article:redis" "article:go"
TOPK.ADD news:trending "article:redis" "article:kubernetes" "article:redis"

# List all Top-5 items
TOPK.LIST news:trending
# 1) "article:redis"
# 2) "article:python"
# 3) "article:docker"
# 4) "article:go"
# 5) "article:kubernetes"
```

## Using WITHCOUNT

The `WITHCOUNT` option returns items paired with their estimated counts:

```bash
TOPK.LIST news:trending WITHCOUNT
# 1) "article:redis"
# 2) (integer) 5
# 3) "article:python"
# 4) (integer) 3
# 5) "article:docker"
# 6) (integer) 1
# 7) "article:go"
# 8) (integer) 1
# 9) "article:kubernetes"
# 10) (integer) 1
```

Note: the counts are approximate estimates, not exact values.

## Building a Trending Section

```bash
TOPK.RESERVE search:queries 10

# Simulate search events over time
TOPK.ADD search:queries "redis tutorial" "python list" "docker compose"
TOPK.ADD search:queries "redis tutorial" "kubernetes ingress" "redis tutorial"
TOPK.ADD search:queries "python list" "redis tutorial" "nginx config"
TOPK.ADD search:queries "redis tutorial" "python list" "aws lambda"

# Get trending searches for homepage
TOPK.LIST search:queries
```

## Python Example: Trending Dashboard

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def get_trending_with_counts(key: str) -> list:
    """Get Top-K items with their estimated counts as a list of dicts."""
    result = r.execute_command("TOPK.LIST", key, "WITHCOUNT")
    if not result:
        return []

    items = []
    for i in range(0, len(result), 2):
        item = result[i]
        count = result[i + 1] if i + 1 < len(result) else 0
        if item:  # Filter out empty slots
            items.append({"item": item, "count": int(count)})
    return items

def get_trending_items(key: str) -> list:
    """Get Top-K items without counts."""
    result = r.execute_command("TOPK.LIST", key)
    return [item for item in result if item]

# Setup
r.execute_command("TOPK.RESERVE", "products:trending", 5)
events = [
    ("laptop", 20), ("headphones", 15), ("mouse", 8),
    ("keyboard", 6), ("monitor", 12), ("webcam", 3)
]
for product, count in events:
    for _ in range(count):
        r.execute_command("TOPK.ADD", "products:trending", product)

# Get trending
trending = get_trending_with_counts("products:trending")
print("Trending Products:")
for i, item in enumerate(trending, 1):
    print(f"  {i}. {item['item']} (~{item['count']} events)")
```

## Node.js Example: API Endpoint

```javascript
const { createClient } = require("redis");

async function getTrendingItems(key, limit = 10) {
    const client = createClient();
    await client.connect();

    const result = await client.sendCommand(["TOPK.LIST", key, "WITHCOUNT"]);

    const items = [];
    for (let i = 0; i < result.length; i += 2) {
        if (result[i]) {
            items.push({
                name: result[i],
                score: parseInt(result[i + 1]) || 0
            });
        }
    }

    await client.disconnect();
    return items.slice(0, limit);
}

const trending = await getTrendingItems("news:trending");
console.log(JSON.stringify(trending, null, 2));
```

## Handling Empty Slots in the List

When a Top-K has not yet seen K distinct items, some slots may be empty (nil):

```bash
TOPK.RESERVE trending 10
TOPK.ADD trending "item1" "item2" "item3"

TOPK.LIST trending
# 1) "item1"
# 2) "item2"
# 3) "item3"
# 4) (nil)
# 5) (nil)
# ... (nil for remaining slots)
```

Always filter nil values in your application:

```python
trending = r.execute_command("TOPK.LIST", "trending")
active_items = [item for item in trending if item is not None]
```

## Comparing Top-K Across Time Windows

```python
def compare_trending_windows(key_hour1: str, key_hour2: str) -> dict:
    """Compare trending items between two time windows."""
    items_h1 = set(get_trending_items(key_hour1))
    items_h2 = set(get_trending_items(key_hour2))

    return {
        "appeared": list(items_h2 - items_h1),    # New in hour 2
        "disappeared": list(items_h1 - items_h2),  # Dropped from hour 1
        "stable": list(items_h1 & items_h2)         # Trending in both
    }
```

## Summary

`TOPK.LIST` retrieves all items currently in a Redis Top-K structure. The `WITHCOUNT` option returns estimated frequency counts alongside each item. Use it to build trending sections, leaderboards, and popular content lists. Filter nil values from the results when the structure has not yet accumulated K distinct items. For checking whether specific items are trending, use `TOPK.QUERY` instead.
