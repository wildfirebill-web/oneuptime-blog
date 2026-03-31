# How to Use FT.SUGDEL in Redis to Remove Suggestions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RediSearch, Autocomplete

Description: Learn how to use FT.SUGDEL in Redis to remove a specific string from an autocomplete suggestion dictionary to keep results accurate.

---

`FT.SUGDEL` removes a specific suggestion string from a RediSearch suggestion dictionary. This is essential for keeping your autocomplete results accurate - removing discontinued products, deleted content, or inappropriate entries from appearing in suggestions.

## Syntax

```text
FT.SUGDEL <key> <string>
```

- `key`: the suggestion dictionary key
- `string`: the exact suggestion string to remove (case-sensitive)

## Return Value

Returns `1` if the suggestion was found and deleted, `0` if the suggestion was not found.

## Basic Example

```bash
# Populate a suggestion dictionary
FT.SUGADD product:suggest "Apple MacBook Pro" 1.0
FT.SUGADD product:suggest "Apple MacBook Air" 0.9
FT.SUGADD product:suggest "Dell XPS 15" 0.8

# Verify suggestions exist
FT.SUGGET product:suggest "apple" MAX 10

# Product discontinued - remove it
FT.SUGDEL product:suggest "Apple MacBook Air"

# Verify it's gone
FT.SUGGET product:suggest "apple" MAX 10
```

After deletion, only "Apple MacBook Pro" appears for the "apple" prefix.

## Python Example: Removing Suggestions on Delete

```python
import redis

r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

def remove_product_suggestion(product_name: str) -> bool:
    """Remove a product from autocomplete when it is deleted."""
    result = r.execute_command("FT.SUGDEL", "product:suggest", product_name)
    removed = bool(result)
    if removed:
        print(f"Removed '{product_name}' from suggestions")
    else:
        print(f"'{product_name}' was not found in suggestions")
    return removed

# Example usage
remove_product_suggestion("Apple MacBook Air")
```

## Handling Case Sensitivity

`FT.SUGDEL` is case-sensitive. The string you pass must exactly match what was added with `FT.SUGADD`:

```bash
FT.SUGADD search:suggest "Python Tutorial" 1.0

# This will return 0 (not found) - case mismatch
FT.SUGDEL search:suggest "python tutorial"

# This will return 1 (deleted) - exact match
FT.SUGDEL search:suggest "Python Tutorial"
```

Store suggestions in a consistent case (e.g., title case or lowercase) to avoid confusion.

## Bulk Removal Pattern

```python
import redis

r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

def bulk_remove_suggestions(key: str, strings: list):
    """Remove multiple suggestions in a pipeline."""
    pipe = r.pipeline()
    for s in strings:
        pipe.execute_command("FT.SUGDEL", key, s)
    results = pipe.execute()

    removed = sum(1 for r in results if r == 1)
    not_found = sum(1 for r in results if r == 0)
    print(f"Removed: {removed}, Not found: {not_found}")

discontinued = [
    "Apple MacBook Air M1",
    "Dell Inspiron 13",
    "HP Spectre x360 13"
]
bulk_remove_suggestions("product:suggest", discontinued)
```

## After Removal: Checking Dictionary Size

```bash
# Check total suggestions remaining
FT.SUGLEN product:suggest
```

## When the Entire Dictionary Should Be Cleared

If you need to rebuild the suggestion dictionary from scratch, delete the key entirely:

```bash
DEL product:suggest
```

Then repopulate with `FT.SUGADD`. This is faster than calling `FT.SUGDEL` individually when removing a large number of entries.

## Summary

`FT.SUGDEL` provides precise, single-entry deletion from a RediSearch suggestion dictionary. Use it when individual items become invalid - discontinued products, deleted articles, or outdated entries. For bulk cleanup, pipeline multiple `FT.SUGDEL` calls or simply delete and rebuild the dictionary key.
