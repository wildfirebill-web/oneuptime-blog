# How to Use VRANGE in Redis to Range Query Vectors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Vector, VRANGE, Vector Set, Search

Description: Learn how to use VRANGE in Redis to retrieve vector elements in rank order from a vector set, with examples for pagination and top-N retrieval.

---

Redis vector sets store elements indexed by their similarity connections in a hierarchical navigable small world (HNSW) graph. The `VRANGE` command lets you retrieve elements ranked by their internal ordering - useful for paginating through vector sets or fetching a subset of entries.

## Basic Syntax

```text
VRANGE key start stop [WITHSCORES]
```

- `start` and `stop` are zero-based indexes (0 = first element)
- Negative indexes count from the end (-1 = last element)
- `WITHSCORES` returns similarity scores alongside element names

## Adding Vectors and Using VRANGE

```bash
# Build a small product vector set
VADD products 0.1 0.2 0.9 laptop
VADD products 0.8 0.1 0.3 phone
VADD products 0.4 0.6 0.5 tablet
VADD products 0.2 0.8 0.4 monitor
VADD products 0.7 0.3 0.6 keyboard

# Retrieve all elements
VRANGE products 0 -1

# Retrieve first 3 elements
VRANGE products 0 2

# Retrieve last 2 elements
VRANGE products -2 -1
```

## Retrieving with Scores

```bash
# Get elements with their internal scores
VRANGE products 0 -1 WITHSCORES
# Returns interleaved: element, score, element, score, ...
```

## Practical Example: Paginating Through a Vector Set

```python
import redis

r = redis.Redis(host="localhost", port=6379)

def paginate_vector_set(key, page=0, page_size=10):
    start = page * page_size
    stop = start + page_size - 1
    results = r.execute_command("VRANGE", key, start, stop)
    return [item.decode() for item in results]

# Page 0: items 0-9
page_0 = paginate_vector_set("products", page=0, page_size=3)
print("Page 0:", page_0)

# Page 1: items 3-5
page_1 = paginate_vector_set("products", page=1, page_size=3)
print("Page 1:", page_1)
```

## Counting Elements Before Ranging

Use `VCARD` to get total count so you can build proper pagination:

```python
def get_all_pages(key, page_size=10):
    total = r.execute_command("VCARD", key)
    pages = []
    for page in range(0, total, page_size):
        stop = min(page + page_size - 1, total - 1)
        batch = r.execute_command("VRANGE", key, page, stop)
        pages.append([item.decode() for item in batch])
    return pages
```

## Combining VRANGE with VSIM

A common pattern is to use `VRANGE` to sample elements and then run `VSIM` against each to find similar items:

```python
def find_cluster_representatives(key, sample_size=5):
    # Get a sample of elements
    samples = r.execute_command("VRANGE", key, 0, sample_size - 1)
    representatives = []
    for elem in samples:
        name = elem.decode()
        # Find the top-3 most similar for each sample
        similar = r.execute_command("VSIM", key, "ELE", name, "COUNT", 3)
        representatives.append({
            "element": name,
            "neighbors": [s.decode() for s in similar]
        })
    return representatives
```

## Error Handling

```bash
# VRANGE on a non-existent key returns empty list
VRANGE nonexistent 0 -1
# Returns: (empty array)

# Out-of-range indexes are clamped silently
VRANGE products 0 1000
# Returns all elements without error
```

## Summary

`VRANGE` provides index-based access to elements in a Redis vector set, complementing similarity-based queries with ordered retrieval. It is ideal for building paginated APIs, sampling subsets for inspection, or bridging vector sets with traditional list-style navigation patterns.
