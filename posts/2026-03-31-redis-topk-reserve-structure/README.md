# How to Use TOPK.RESERVE in Redis to Create a Top-K Structure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Top-K, Probabilistic Data Structure

Description: Learn how to use TOPK.RESERVE in Redis to create a Top-K data structure that tracks the most frequent items in a stream with configurable accuracy settings.

---

`TOPK.RESERVE` initializes a Top-K data structure in Redis. This probabilistic data structure efficiently tracks the K most frequently occurring items in a data stream without storing all items - making it ideal for finding trending items, top users, most-visited URLs, and similar frequency ranking tasks.

## Basic Syntax

```text
TOPK.RESERVE key topk [width depth decay]
```

- `topk` - Number of top items to track
- `width` - Number of counters per array (default: 8) - higher = more accurate but uses more memory
- `depth` - Number of arrays (default: 7) - higher = lower probability of losing top items
- `decay` - Decay factor for older counts (default: 0.9) - value between 0 and 1

## Creating a Basic Top-K Structure

```bash
# Track top 10 items with default accuracy settings
127.0.0.1:6379> TOPK.RESERVE trending:products 10
OK
```

## Creating a High-Accuracy Top-K Structure

```bash
# Track top 20 items with higher accuracy (more memory)
127.0.0.1:6379> TOPK.RESERVE popular:pages 20 50 5 0.9
OK
```

Increasing `width` and `depth` improves accuracy at the cost of memory. The default settings work well for most use cases.

## Python Example: Setting Up Top-K for Different Scenarios

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def create_topk(key: str, k: int,
                width: int = 8,
                depth: int = 7,
                decay: float = 0.9) -> None:
    """Create a Top-K structure with the given parameters."""
    try:
        if width == 8 and depth == 7 and decay == 0.9:
            # Use defaults
            r.execute_command("TOPK.RESERVE", key, k)
        else:
            r.execute_command("TOPK.RESERVE", key, k, width, depth, decay)
        print(f"Created Top-{k} structure: '{key}'")
    except redis.ResponseError as e:
        if "already exists" in str(e).lower():
            print(f"Structure '{key}' already exists")
        else:
            raise

# Top 10 trending hashtags (default accuracy)
create_topk("trending:hashtags", k=10)

# Top 5 most-visited pages (higher accuracy)
create_topk("popular:pages", k=5, width=50, depth=5)

# Top 100 active users (default, large k)
create_topk("active:users", k=100)
```

## Adding Items After Creation

After reserving the structure, use `TOPK.ADD` to insert items:

```bash
127.0.0.1:6379> TOPK.ADD trending:products "Widget A" "Gadget B" "Widget A" "Widget A"
1) (nil)
2) (nil)
3) (nil)
4) (nil)
```

`nil` means no item was displaced from the top K. When an item displaces another, its key is returned.

## Choosing the Right Parameters

```python
# Memory vs accuracy tradeoff guide:
configs = {
    "minimal":    {"width": 8,  "depth": 7, "note": "~1KB, suitable for prototyping"},
    "balanced":   {"width": 20, "depth": 7, "note": "~5KB, good for most production use"},
    "accurate":   {"width": 50, "depth": 5, "note": "~10KB, high accuracy for critical tracking"},
    "high-volume":{"width": 8,  "depth": 7, "note": "default but with larger k value"},
}

for name, config in configs.items():
    print(f"{name}: width={config['width']}, depth={config['depth']} - {config['note']}")
```

## Verifying the Structure

```bash
127.0.0.1:6379> TOPK.INFO trending:products
1) k
2) (integer) 10
3) width
4) (integer) 8
5) depth
6) (integer) 7
7) decay
8) "0.9"
```

## Complete Setup Example

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

# Create a Top-5 leaderboard for game scores
r.execute_command("TOPK.RESERVE", "game:leaderboard", 5, 20, 7, 0.9)

# Simulate player activity
player_actions = [
    "player:alice", "player:bob", "player:alice",
    "player:charlie", "player:alice", "player:bob",
    "player:dave", "player:alice", "player:eve",
]
for player in player_actions:
    r.execute_command("TOPK.ADD", "game:leaderboard", player)

# Get the top players
top_players = r.execute_command("TOPK.LIST", "game:leaderboard")
print(f"Top players: {top_players}")
```

## Summary

`TOPK.RESERVE` initializes a Top-K structure in Redis with configurable accuracy parameters. By adjusting `width`, `depth`, and `decay`, you can balance memory usage against tracking accuracy. This structure is the foundation for real-time leaderboards, trending item detection, and frequency-ranked analytics in high-throughput systems.
