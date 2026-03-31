# How to Use FT.SUGGET in Redis to Retrieve Suggestions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RediSearch, Autocomplete

Description: Learn how to use FT.SUGGET in Redis to query an autocomplete suggestion dictionary with fuzzy matching, scores, and payloads.

---

`FT.SUGGET` retrieves autocomplete suggestions from a RediSearch suggestion dictionary. It supports prefix matching, fuzzy matching for typo tolerance, and optional return of scores and payloads. This is the query side of the `FT.SUGADD`/`FT.SUGGET` autocomplete pair.

## Syntax

```text
FT.SUGGET <key> <prefix> [FUZZY] [WITHSCORES] [WITHPAYLOADS] [MAX <num>]
```

- `key`: the suggestion dictionary key
- `prefix`: the prefix string to search for
- `FUZZY`: enable fuzzy (Levenshtein distance 1) matching for typo tolerance
- `WITHSCORES`: include suggestion scores in the response
- `WITHPAYLOADS`: include payloads attached when suggestions were added
- `MAX <num>`: limit the number of results (default is 5)

## Populating a Dictionary First

```bash
FT.SUGADD city:suggest "New York" 1.0 PAYLOAD "US-NY"
FT.SUGADD city:suggest "New Orleans" 0.9 PAYLOAD "US-LA"
FT.SUGADD city:suggest "Newark" 0.8 PAYLOAD "US-NJ"
FT.SUGADD city:suggest "London" 1.0 PAYLOAD "GB-LND"
FT.SUGADD city:suggest "Los Angeles" 0.95 PAYLOAD "US-CA"
```

## Basic Prefix Search

```bash
FT.SUGGET city:suggest "New" MAX 10
```

Response:

```text
1) "New York"
2) "New Orleans"
3) "Newark"
```

## Fuzzy Matching for Typos

With `FUZZY`, Redis returns suggestions within edit distance 1:

```bash
FT.SUGGET city:suggest "Londn" FUZZY MAX 5
```

Response:

```text
1) "London"
```

Without `FUZZY`, a misspelling returns nothing.

## Returning Scores and Payloads

```bash
FT.SUGGET city:suggest "New" WITHSCORES WITHPAYLOADS MAX 5
```

Response (score, payload interleaved per result):

```text
1) "New York"
2) "1.0"
3) "US-NY"
4) "New Orleans"
5) "0.9"
6) "US-LA"
7) "Newark"
8) "0.8"
9) "US-NJ"
```

## Python: Building a Typeahead API

```python
import redis

r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

def typeahead(prefix: str, fuzzy: bool = True, max_results: int = 5):
    args = ["FT.SUGGET", "city:suggest", prefix]
    if fuzzy:
        args.append("FUZZY")
    args += ["WITHPAYLOADS", "MAX", max_results]

    raw = r.execute_command(*args)

    # Parse alternating name/payload pairs
    results = []
    for i in range(0, len(raw), 2):
        results.append({
            "name": raw[i],
            "id": raw[i + 1]
        })
    return results

# Test it
suggestions = typeahead("new")
for s in suggestions:
    print(f"{s['name']} ({s['id']})")
```

## Performance Considerations

- The suggestion dictionary is stored as an in-memory trie - queries are very fast (sub-millisecond)
- For large dictionaries (millions of entries), prefix queries still return results in microseconds
- `FUZZY` adds slight overhead due to edit-distance computation but is still fast

## Combining with FT.SEARCH

Use `FT.SUGGET` for fast prefix hints in the UI, and `FT.SEARCH` for the full search results page:

```python
# As user types - fast prefix hints
hints = r.execute_command("FT.SUGGET", "product:suggest", user_input, "FUZZY", "MAX", "5")

# On search submit - full ranked results
results = r.execute_command("FT.SEARCH", "products", f"@name:{user_input}*")
```

## Summary

`FT.SUGGET` queries a RediSearch suggestion dictionary by prefix, with optional fuzzy matching for typo tolerance. It returns suggestions ranked by score, and can include metadata payloads added via `FT.SUGADD`. This combination makes it a practical building block for low-latency, typo-tolerant autocomplete features in Redis.
