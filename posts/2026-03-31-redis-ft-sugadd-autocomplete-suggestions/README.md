# How to Use FT.SUGADD in Redis for Autocomplete Suggestions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RediSearch, Autocomplete

Description: Learn how to use FT.SUGADD in Redis to build an autocomplete suggestion dictionary with optional payloads and scoring weights.

---

`FT.SUGADD` adds a string to a RediSearch suggestion dictionary. These dictionaries power autocomplete features - when a user types a few characters, `FT.SUGGET` returns ranked suggestions from the dictionary. `FT.SUGADD` is how you populate and maintain that dictionary.

## Syntax

```text
FT.SUGADD <key> <string> <score> [INCR] [PAYLOAD <payload>]
```

- `key`: the suggestion dictionary key
- `string`: the suggestion string to add
- `score`: a floating-point relevance score (higher = more relevant)
- `INCR`: if present, increment the existing score instead of replacing it
- `PAYLOAD`: optional string payload returned with the suggestion (e.g., metadata or an ID)

## Basic Example: Product Autocomplete

```bash
FT.SUGADD product:suggest "Apple MacBook Pro" 1.0
FT.SUGADD product:suggest "Apple MacBook Air" 0.9
FT.SUGADD product:suggest "Apple iPad Pro" 0.8
FT.SUGADD product:suggest "Dell XPS 15" 0.7
FT.SUGADD product:suggest "Dell Inspiron 14" 0.6
```

Now retrieve suggestions when a user types "apple":

```bash
FT.SUGGET product:suggest "apple" FUZZY MAX 5
```

Response:

```text
1) "Apple MacBook Pro"
2) "Apple MacBook Air"
3) "Apple iPad Pro"
```

## Using INCR for Click-Based Scoring

Boost relevance when users click on a suggestion:

```bash
# Every time "Apple MacBook Pro" is clicked, increase its score
FT.SUGADD product:suggest "Apple MacBook Pro" 1.0 INCR
```

This makes popular suggestions rise to the top organically over time.

## Adding Payloads for Richer Responses

Attach metadata to each suggestion, like a product ID:

```bash
FT.SUGADD product:suggest "Apple MacBook Pro" 1.0 PAYLOAD "prod:1001"
FT.SUGADD product:suggest "Apple MacBook Air" 0.9 PAYLOAD "prod:1002"
```

Retrieve with payloads:

```bash
FT.SUGGET product:suggest "apple" WITHPAYLOADS MAX 5
```

Response includes payloads:

```text
1) "Apple MacBook Pro"
2) "prod:1001"
3) "Apple MacBook Air"
4) "prod:1002"
```

## Python Example: Bulk Loading Suggestions

```python
import redis

r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

products = [
    ("Apple MacBook Pro 14", 1.0, "prod:1001"),
    ("Apple MacBook Air M2", 0.95, "prod:1002"),
    ("Apple iPad Pro 12.9", 0.9, "prod:1003"),
    ("Dell XPS 15", 0.85, "prod:1004"),
    ("Dell Inspiron 14", 0.8, "prod:1005"),
]

pipeline = r.pipeline()
for name, score, payload in products:
    pipeline.execute_command(
        "FT.SUGADD", "product:suggest", name, score, "PAYLOAD", payload
    )
pipeline.execute()
print(f"Added {len(products)} suggestions")
```

## Checking Dictionary Size

```bash
FT.SUGLEN product:suggest
```

## Building Search-as-You-Type UI

Combine `FT.SUGADD` with `FT.SUGGET` for a complete search-as-you-type experience:

```python
def add_suggestion(key: str, text: str, score: float = 1.0):
    r.execute_command("FT.SUGADD", key, text, score)

def get_suggestions(key: str, prefix: str, count: int = 5):
    return r.execute_command("FT.SUGGET", key, prefix, "FUZZY", "MAX", count)

# Add on product creation
add_suggestion("product:suggest", "Apple MacBook Pro", 1.0)

# Query on keypress
suggestions = get_suggestions("product:suggest", "mac")
```

## Summary

`FT.SUGADD` is the core command for populating RediSearch autocomplete dictionaries. It supports weighted scoring, score increments for popularity-based ranking, and optional payloads for attaching metadata. Combined with `FT.SUGGET`, it enables fast, typo-tolerant autocomplete features backed by an in-memory trie structure.
