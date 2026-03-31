# How to Use CF.MEXISTS in Redis to Check Multiple Cuckoo Filter Items

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cuckoo Filter, Probabilistic Data Structure

Description: Learn how to use CF.MEXISTS in Redis to check membership of multiple items in a Cuckoo filter in a single command, reducing round-trips and latency.

---

`CF.MEXISTS` checks whether multiple items are likely present in a Redis Cuckoo filter in one command. This is the batch equivalent of `CF.EXISTS` and is essential for performance-sensitive lookups where you need to check many items at once without issuing individual queries.

## Basic Syntax

```text
CF.MEXISTS key item [item ...]
```

Returns an array of integers: `1` if the item is likely present, `0` if it is definitely not present.

## Basic Example

```bash
# Setup
127.0.0.1:6379> CF.RESERVE email:blocklist 100000
OK
127.0.0.1:6379> CF.ADD email:blocklist "spam@evil.com"
(integer) 1
127.0.0.1:6379> CF.ADD email:blocklist "phish@bad.com"
(integer) 1

# Check multiple items
127.0.0.1:6379> CF.MEXISTS email:blocklist "spam@evil.com" "legit@good.com" "phish@bad.com"
1) (integer) 1
2) (integer) 0
3) (integer) 1
```

`legit@good.com` returns `0` - definitely not in the filter.

## Python Example: Batch Membership Check

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def check_blocklist(emails: list, filter_key: str = "email:blocklist") -> dict:
    """
    Check a list of emails against the Cuckoo filter blocklist.
    Returns a dict mapping email -> is_blocked (bool).
    """
    if not emails:
        return {}
    results = r.execute_command("CF.MEXISTS", filter_key, *emails)
    return {email: bool(result) for email, result in zip(emails, results)}

# Check a batch of emails
emails = [
    "user@legit.com",
    "spam@evil.com",
    "admin@company.com",
    "phish@bad.com",
]

status = check_blocklist(emails)
for email, blocked in status.items():
    label = "BLOCKED" if blocked else "OK"
    print(f"{label}: {email}")
```

## Filtering Out Known Items

A common use case is to filter a batch down to only genuinely new items:

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def get_unseen_items(items: list, filter_key: str) -> list:
    """Return only items that are definitely not in the filter."""
    results = r.execute_command("CF.MEXISTS", filter_key, *items)
    return [item for item, seen in zip(items, results) if seen == 0]

# Setup filter with some existing items
r.execute_command("CF.RESERVE", "product:seen", 50000)
r.execute_command("CF.ADD", "product:seen", "PROD-001")
r.execute_command("CF.ADD", "product:seen", "PROD-002")

incoming = ["PROD-001", "PROD-003", "PROD-002", "PROD-004"]
new_products = get_unseen_items(incoming, "product:seen")
print(f"New products to process: {new_products}")
# Output: New products to process: ['PROD-003', 'PROD-004']
```

## CF.MEXISTS vs CF.EXISTS

| Feature | CF.EXISTS | CF.MEXISTS |
|---|---|---|
| Items per call | 1 | Multiple |
| Round-trips for N items | N | 1 |
| Return type | Integer | Array of integers |

For checking 1000 items, `CF.MEXISTS` with all items in one call is significantly faster than 1000 individual `CF.EXISTS` calls.

## Understanding False Positives

```python
# CF.MEXISTS can return false positives (1 for items not actually inserted)
# It will NEVER return a false negative (0 for items that ARE present)

# Safe to use: if result is 0, item is DEFINITELY not in the filter
# Caution: if result is 1, item is PROBABLY in the filter (small false positive rate)

results = r.execute_command("CF.MEXISTS", "email:blocklist",
                             "definitely-not-added@new.com")
# Result is 0 with very high probability - item is not in filter
```

## Summary

`CF.MEXISTS` is the efficient batch membership check command for Redis Cuckoo filters. By checking multiple items in a single network round-trip, it dramatically reduces latency compared to individual `CF.EXISTS` calls. It's ideal for filtering pipelines, blocklist enforcement, and deduplication workflows where you need to classify many items at once.
