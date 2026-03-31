# How to Use ZRANGE in Redis to Retrieve Sorted Set Ranges

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Sorted Sets, ZRANGE, Range Queries, Commands

Description: Learn how to use the unified ZRANGE command in Redis 6.2+ to retrieve sorted set members by index, score, or lexicographic range in any direction.

---

## What Is ZRANGE

`ZRANGE` is the unified range retrieval command introduced in Redis 6.2. It replaces and combines `ZRANGEBYSCORE`, `ZRANGEBYLEX`, `ZREVRANGEBYSCORE`, `ZREVRANGEBYLEX`, and `ZREVRANGE` into a single flexible command with a `REV`, `BYSCORE`, and `BYLEX` options.

## Syntax

```text
ZRANGE key min max [BYSCORE|BYLEX] [REV] [LIMIT offset count] [WITHSCORES]
```

- `key` - the sorted set key
- `min`, `max` - range bounds (index, score, or lex depending on options)
- `BYSCORE` - interpret `min`/`max` as score values
- `BYLEX` - interpret `min`/`max` as lexicographic values
- `REV` - reverse the iteration direction
- `LIMIT offset count` - pagination (only with `BYSCORE` or `BYLEX`)
- `WITHSCORES` - include scores in response

## Basic Index-Based Range

### Get All Members

```bash
redis-cli ZADD leaderboard 100 "alice" 200 "bob" 300 "charlie" 400 "dave"

redis-cli ZRANGE leaderboard 0 -1
```

```text
1) "alice"
2) "bob"
3) "charlie"
4) "dave"
```

### Get with Scores

```bash
redis-cli ZRANGE leaderboard 0 -1 WITHSCORES
```

```text
1) "alice"
2) "100"
3) "bob"
4) "200"
5) "charlie"
6) "300"
7) "dave"
8) "400"
```

### Get Specific Slice

```bash
# First 2 elements
redis-cli ZRANGE leaderboard 0 1 WITHSCORES
```

```text
1) "alice"
2) "100"
3) "bob"
4) "200"
```

### Reverse Order

```bash
redis-cli ZRANGE leaderboard 0 -1 REV
```

```text
1) "dave"
2) "charlie"
3) "bob"
4) "alice"
```

## Score-Based Range (BYSCORE)

```bash
redis-cli ZRANGE leaderboard 150 350 BYSCORE WITHSCORES
```

```text
1) "bob"
2) "200"
3) "charlie"
4) "300"
```

### With -inf and +inf

```bash
redis-cli ZRANGE leaderboard -inf +inf BYSCORE WITHSCORES
```

### Exclusive Bounds

```bash
# Exclude 200 from lower bound
redis-cli ZRANGE leaderboard "(200" +inf BYSCORE WITHSCORES
```

```text
1) "charlie"
2) "300"
3) "dave"
4) "400"
```

### Descending Score Range

When using `REV` with `BYSCORE`, swap min and max:

```bash
redis-cli ZRANGE leaderboard +inf 150 BYSCORE REV WITHSCORES
```

```text
1) "dave"
2) "400"
3) "charlie"
4) "300"
5) "bob"
6) "200"
```

### Pagination with LIMIT

```bash
# Page 1: first 2 results above score 100
redis-cli ZRANGE leaderboard 100 +inf BYSCORE LIMIT 0 2 WITHSCORES

# Page 2: next 2 results
redis-cli ZRANGE leaderboard 100 +inf BYSCORE LIMIT 2 2 WITHSCORES
```

## Lexicographic Range (BYLEX)

Only meaningful when all members share the same score.

```bash
redis-cli ZADD words 0 "apple" 0 "apricot" 0 "banana" 0 "blueberry" 0 "cherry"

redis-cli ZRANGE words "[a" "(b" BYLEX
```

```text
1) "apple"
2) "apricot"
```

### All Words from "b" Onwards

```bash
redis-cli ZRANGE words "[b" "+" BYLEX
```

```text
1) "banana"
2) "blueberry"
3) "cherry"
```

## Practical Examples

### Top N Leaderboard

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

r.zadd('scores', {'alice': 9500, 'bob': 8200, 'charlie': 9800, 'dave': 7100, 'eve': 9200})

def get_top_n(n):
    results = r.zrange('scores', 0, n-1, desc=True, withscores=True)
    return [(member, int(score)) for member, score in results]

print("Top 3:", get_top_n(3))
# [('charlie', 9800), ('alice', 9500), ('eve', 9200)]
```

### Paginated Score Query

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

r.zadd('products', {f'prod:{i}': i * 10 for i in range(1, 21)})

def get_products_page(min_price, max_price, page=0, page_size=5):
    offset = page * page_size
    results = r.zrange('products', min_price, max_price,
                        byscore=True, withscores=True,
                        offset=offset, count=page_size)
    return results

page1 = get_products_page(0, 200, page=0)
print(f"Page 1: {page1}")
page2 = get_products_page(0, 200, page=1)
print(f"Page 2: {page2}")
```

### Autocomplete with Lexicographic Range

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

items = ['python', 'pytorch', 'postgresql', 'prometheus', 'rabbitmq', 'redis', 'rust']
r.zadd('search_index', {item: 0 for item in items})

def autocomplete(prefix, limit=5):
    upper = prefix[:-1] + chr(ord(prefix[-1]) + 1)
    results = r.zrange('search_index', f'[{prefix}', f'({upper}',
                        bylex=True)
    return results[:limit]

print(autocomplete('p'))  # ['postgresql', 'prometheus', 'python', 'pytorch']
print(autocomplete('py')) # ['python', 'pytorch']
```

## Summary

`ZRANGE` in Redis 6.2+ is the single unified command for all sorted set range queries, supporting index-based, score-based, and lexicographic retrieval in both ascending and descending order with built-in pagination. It replaces multiple older commands (`ZRANGEBYSCORE`, `ZREVRANGE`, etc.) and should be used for all new code. When using `REV` with `BYSCORE`, remember to swap `min` and `max` so the higher value comes first.
