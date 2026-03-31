# How to Use LPOS in Redis to Find Element Positions in a List

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Lists, LPOS, Search, Commands

Description: Learn how to use the LPOS command in Redis to find the position of one or more occurrences of an element within a list, with rank and count options.

---

## What Is LPOS

`LPOS` returns the index (zero-based position) of a matching element within a Redis list. Unlike `LRANGE` which requires you to scan the whole list client-side, `LPOS` does the search server-side and supports finding multiple occurrences.

It was introduced in Redis 6.0.6 as a replacement for the slower pattern of `LRANGE` + client-side search.

## Syntax

```text
LPOS key element [RANK rank] [COUNT num-matches] [MAXLEN len]
```

- `key` - the list key
- `element` - the value to search for
- `RANK rank` - skip the first `rank-1` matches (use negative rank to search from the tail)
- `COUNT num-matches` - return up to this many positions; `0` returns all
- `MAXLEN len` - limit search to this many elements from the head (or tail if RANK is negative); `0` scans the whole list

Returns the zero-based index, or nil if not found. Returns an array when COUNT is specified.

## Basic Usage

### Find First Occurrence

```bash
redis-cli RPUSH mylist "a" "b" "c" "b" "d" "b"
redis-cli LPOS mylist "b"
```

```text
(integer) 1
```

### Element Not Found

```bash
redis-cli LPOS mylist "z"
```

```text
(nil)
```

### Find All Occurrences

Use `COUNT 0` to return positions of all matches:

```bash
redis-cli LPOS mylist "b" COUNT 0
```

```text
1) (integer) 1
2) (integer) 3
3) (integer) 5
```

### Limit Number of Matches

```bash
redis-cli LPOS mylist "b" COUNT 2
```

```text
1) (integer) 1
2) (integer) 3
```

## Using RANK to Skip Occurrences

### Skip the First Match

`RANK 2` means return the second match:

```bash
redis-cli LPOS mylist "b" RANK 2
```

```text
(integer) 3
```

### Search from the Tail

Use negative RANK to search from the right end of the list. `RANK -1` returns the last match:

```bash
redis-cli LPOS mylist "b" RANK -1
```

```text
(integer) 5
```

`RANK -2` returns the second-to-last match:

```bash
redis-cli LPOS mylist "b" RANK -2
```

```text
(integer) 3
```

## Using MAXLEN to Limit Search Scope

`MAXLEN` caps how many elements are scanned, useful for performance on large lists:

```bash
# Only search the first 3 elements
redis-cli LPOS mylist "b" MAXLEN 3
```

```text
(integer) 1
```

```bash
# Search first 3 elements for "d" - not found there
redis-cli LPOS mylist "d" MAXLEN 3
```

```text
(nil)
```

## Practical Examples in Python

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Setup
r.delete('events')
r.rpush('events', 'login', 'click', 'login', 'purchase', 'click', 'login', 'logout')

# Find first login
pos = r.lpos('events', 'login')
print(f"First login at index: {pos}")  # 0

# Find all login positions
all_logins = r.lpos('events', 'login', count=0)
print(f"All login positions: {all_logins}")  # [0, 2, 5]

# Find last login
last_login = r.lpos('events', 'login', rank=-1)
print(f"Last login at index: {last_login}")  # 5

# Find second login
second_login = r.lpos('events', 'login', rank=2)
print(f"Second login at index: {second_login}")  # 2
```

## Use Case - Finding Duplicate Entries

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def find_duplicates(key, element):
    """Return True if element appears more than once in list."""
    positions = r.lpos(key, element, count=2)
    return len(positions) > 1

def get_element_count(key, element):
    """Count occurrences of element in list."""
    positions = r.lpos(key, element, count=0)
    return len(positions) if positions else 0

r.delete('log')
r.rpush('log', 'error:1', 'warning:1', 'error:1', 'info:1', 'error:1')

print(f"Error duplicated: {find_duplicates('log', 'error:1')}")  # True
print(f"Error count: {get_element_count('log', 'error:1')}")     # 3
```

## Use Case - Position-Based Pagination

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def get_context_around(key, element, context=2):
    """Get surrounding elements around a match."""
    pos = r.lpos(key, element)
    if pos is None:
        return None
    start = max(0, pos - context)
    end = pos + context
    items = r.lrange(key, start, end)
    return {'position': pos, 'context': items}

r.delete('timeline')
r.rpush('timeline', 'e1', 'e2', 'e3', 'TARGET', 'e5', 'e6', 'e7')

result = get_context_around('timeline', 'TARGET', context=2)
print(result)
# {'position': 3, 'context': ['e2', 'e3', 'TARGET', 'e5', 'e6']}
```

## Summary

`LPOS` is an efficient server-side search command for Redis lists that returns zero-based positions of matching elements. The `RANK` option enables skipping previous matches or searching from the tail, `COUNT` controls how many positions to return, and `MAXLEN` limits scan depth for performance on large lists. It is particularly useful for deduplication checks, event log analysis, and finding insertion points without fetching entire lists to the client.
