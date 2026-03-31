# How to Choose Between Bloom Filter and Cuckoo Filter in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Bloom Filter, Cuckoo Filter, Probabilistic, Data Structures

Description: Learn the key differences between Redis Bloom and Cuckoo filters and how to choose the right one for your use case based on deletion support and performance.

---

## Overview of Probabilistic Set Filters

Both Bloom and Cuckoo filters answer the question "Is this item in the set?" with possible false positives but zero false negatives. They use far less memory than storing the actual set.

Key difference: Cuckoo filters support deletion; Bloom filters do not.

## Bloom Filter

A Bloom filter uses multiple hash functions mapping each item to positions in a bit array. When testing membership, it checks if all bit positions are set. If any bit is 0, the item is definitely not in the set. If all are 1, the item is probably in the set.

```bash
# Create a Bloom filter for 1 million items with 0.1% false positive rate
BF.RESERVE bloomfilter:emails 0.001 1000000

# Add items
BF.ADD bloomfilter:emails user@example.com

# Add multiple items
BF.MADD bloomfilter:emails a@b.com c@d.com e@f.com

# Test membership
BF.EXISTS bloomfilter:emails user@example.com

# Test multiple at once
BF.MEXISTS bloomfilter:emails a@b.com unknown@x.com
```

## Cuckoo Filter

A Cuckoo filter uses cuckoo hashing to store fingerprints of items. It supports deletion by removing fingerprints and generally has better performance for membership checks than Bloom filters for the same false positive rate.

```bash
# Create a Cuckoo filter for 1 million items
CF.RESERVE cuckoofilter:emails 1000000

# Add items
CF.ADD cuckoofilter:emails user@example.com

# Add only if not present (prevents duplicate insertions)
CF.ADDNX cuckoofilter:emails user@example.com

# Test membership
CF.EXISTS cuckoofilter:emails user@example.com

# Delete an item
CF.DEL cuckoofilter:emails user@example.com
```

## Comparison Table

```text
Feature                  | Bloom Filter        | Cuckoo Filter
-------------------------|---------------------|--------------------
Deletion support         | No                  | Yes
False positive rate      | Configurable        | Configurable
Space efficiency         | Slightly better     | Slightly worse
Insert performance       | O(k) hash ops       | O(1) amortized
Lookup performance       | O(k) hash ops       | O(1)
Duplicate handling       | Transparent         | Requires ADDNX
Best for                 | Append-only sets    | Dynamic sets
```

## Python Implementation

```python
from redis import Redis

r = Redis(decode_responses=True)

# ---- Bloom Filter ----

def init_bloom(key: str, error_rate: float = 0.001, capacity: int = 1_000_000):
    try:
        r.bf().info(key)
    except Exception:
        r.bf().reserve(key, error_rate, capacity)

def bloom_add(key: str, item: str) -> bool:
    return r.bf().add(key, item)

def bloom_exists(key: str, item: str) -> bool:
    return r.bf().exists(key, item)

def bloom_madd(key: str, *items: str) -> list[bool]:
    return r.bf().madd(key, *items)

def bloom_mexists(key: str, *items: str) -> list[bool]:
    return r.bf().mexists(key, *items)

# ---- Cuckoo Filter ----

def init_cuckoo(key: str, capacity: int = 1_000_000):
    try:
        r.cf().info(key)
    except Exception:
        r.cf().reserve(key, capacity)

def cuckoo_add(key: str, item: str) -> bool:
    return r.cf().add(key, item)

def cuckoo_add_if_new(key: str, item: str) -> bool:
    return r.cf().addnx(key, item)

def cuckoo_exists(key: str, item: str) -> bool:
    return r.cf().exists(key, item)

def cuckoo_delete(key: str, item: str) -> bool:
    return r.cf().delete(key, item)
```

## Decision Guide

```python
def choose_filter(needs_deletion: bool, cardinality: str) -> str:
    """
    needs_deletion: True if items need to be removable
    cardinality: 'low' (<100k), 'medium' (100k-10M), 'high' (>10M)
    """
    if needs_deletion:
        return "Cuckoo Filter (CF.*)"

    if cardinality == "high":
        # Bloom uses slightly less memory for very large sets
        return "Bloom Filter (BF.*)"

    # For medium/low cardinality without deletion, either works
    # Cuckoo has better lookup performance
    return "Cuckoo Filter (CF.*) - better lookup performance"
```

## Practical Example - Deduplication

```python
# Use Bloom for append-only deduplication (newsletter signups)
def register_newsletter(email: str) -> bool:
    key = "bloom:newsletter:subscribers"
    init_bloom(key, error_rate=0.0001, capacity=5_000_000)
    if bloom_exists(key, email):
        return False  # Already subscribed (or false positive)
    bloom_add(key, email)
    return True

# Use Cuckoo for active user session tracking (needs deletion on logout)
def user_online(user_id: str):
    cuckoo_add_if_new("cf:online_users", user_id)

def user_offline(user_id: str):
    cuckoo_delete("cf:online_users", user_id)

def is_online(user_id: str) -> bool:
    return cuckoo_exists("cf:online_users", user_id)
```

## Summary

Choose a Bloom filter when you have an append-only set and want the most memory-efficient option. Choose a Cuckoo filter when you need to delete items from the set, as Bloom filters have no deletion support. For most new applications, Cuckoo filters are the better default choice due to their deletion capability and faster lookup performance.
