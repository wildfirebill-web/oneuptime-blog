# How to Use FT.SUGLEN in Redis to Count Suggestions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RediSearch, Autocomplete

Description: Learn how to use FT.SUGLEN in Redis to count the number of entries in an autocomplete suggestion dictionary for monitoring and validation.

---

`FT.SUGLEN` returns the number of entries in a RediSearch suggestion dictionary. It is a quick utility command useful for validating that your autocomplete dictionary has been populated correctly, monitoring dictionary growth, and comparing counts before and after bulk operations.

## Syntax

```text
FT.SUGLEN <key>
```

- `key`: the suggestion dictionary key

Returns an integer - the total count of suggestions in the dictionary.

## Basic Example

```bash
# Create and populate a suggestion dictionary
FT.SUGADD search:suggest "Python Tutorial" 1.0
FT.SUGADD search:suggest "Python Decorators" 0.9
FT.SUGADD search:suggest "Python Async Programming" 0.85
FT.SUGADD search:suggest "JavaScript Promises" 0.8
FT.SUGADD search:suggest "JavaScript Arrow Functions" 0.75

# Check total count
FT.SUGLEN search:suggest
```

Response:

```text
(integer) 5
```

## Checking an Empty or Non-Existent Key

If the key does not exist or the dictionary is empty:

```bash
FT.SUGLEN nonexistent:key
```

Response:

```text
(integer) 0
```

## Python Example: Validating Dictionary Population

```python
import redis

r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

def validate_suggestion_load(key: str, expected_min: int) -> bool:
    """Check that the suggestion dictionary has enough entries."""
    count = r.execute_command("FT.SUGLEN", key)
    print(f"Suggestion dictionary '{key}' has {count} entries")

    if count < expected_min:
        print(f"Warning: expected at least {expected_min} suggestions, got {count}")
        return False
    return True

# After bulk loading suggestions from a product catalog
validate_suggestion_load("product:suggest", 1000)
```

## Monitoring Dictionary Growth

Track suggestion dictionary size over time to detect unexpected drops (e.g., accidental deletion):

```python
import redis
import time

r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

def monitor_suggestion_count(key: str, interval_seconds: int = 60):
    """Periodically log the suggestion dictionary size."""
    while True:
        count = r.execute_command("FT.SUGLEN", key)
        print(f"[{time.strftime('%H:%M:%S')}] {key}: {count} suggestions")
        time.sleep(interval_seconds)
```

## Workflow: Add, Count, Query, Clean

```bash
# Step 1: Bulk load suggestions
FT.SUGADD products:suggest "Laptop" 1.0
FT.SUGADD products:suggest "Laptop Stand" 0.9
FT.SUGADD products:suggest "Laptop Bag" 0.85

# Step 2: Confirm count
FT.SUGLEN products:suggest
# Returns: 3

# Step 3: Query suggestions
FT.SUGGET products:suggest "lapt" MAX 5

# Step 4: Delete one entry
FT.SUGDEL products:suggest "Laptop Bag"

# Step 5: Verify count decreased
FT.SUGLEN products:suggest
# Returns: 2
```

## Comparing Count Before and After Operations

```python
import redis

r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

key = "product:suggest"

before = r.execute_command("FT.SUGLEN", key)

# Bulk delete discontinued items
discontinued = ["Old Product A", "Old Product B", "Old Product C"]
for item in discontinued:
    r.execute_command("FT.SUGDEL", key, item)

after = r.execute_command("FT.SUGLEN", key)
print(f"Removed {before - after} suggestions ({before} -> {after})")
```

## Summary

`FT.SUGLEN` is a lightweight introspection command that returns the total number of entries in a RediSearch suggestion dictionary. Use it to validate bulk loads, monitor dictionary health, and confirm that deletions are working as expected in your autocomplete feature maintenance workflows.
