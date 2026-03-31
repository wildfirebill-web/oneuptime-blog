# How to Use LTRIM in Redis to Trim a List to a Range

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, List, LTRIM, Command

Description: Learn how to use the Redis LTRIM command to trim a list to a specified index range, keeping only the elements you need.

---

## How LTRIM Works

`LTRIM` modifies a list in place, keeping only the elements within a specified index range and discarding everything outside it. It is a destructive operation - elements outside the range are permanently deleted.

LTRIM is most commonly used to cap the size of a list, such as keeping only the last N entries in a log or activity feed. When combined with LPUSH or RPUSH, it creates an efficient capped-list pattern.

```mermaid
graph LR
    A["List: [a, b, c, d, e]"] --> B["LTRIM key 1 3"]
    B --> C["List: [b, c, d]"]
```

## Syntax

```redis
LTRIM key start stop
```

- `key` - the list key
- `start` - start index (inclusive), zero-based; negative indexes count from tail
- `stop` - stop index (inclusive); negative indexes count from tail

Returns `OK`. If the range is entirely out of bounds, the list is emptied (deleted).

## Examples

### Basic Trim

```redis
RPUSH mylist "a" "b" "c" "d" "e"
LTRIM mylist 1 3
LRANGE mylist 0 -1
```

```text
1) "b"
2) "c"
3) "d"
```

Elements at indexes 0 ("a") and 4 ("e") were removed.

### Keep Last N Elements

A common pattern is to keep only the most recent N items. Because LPUSH prepends to the head, the newest items are at the start.

```redis
LPUSH recentviews "page5"
LPUSH recentviews "page4"
LPUSH recentviews "page3"
LPUSH recentviews "page2"
LPUSH recentviews "page1"
LTRIM recentviews 0 2
LRANGE recentviews 0 -1
```

```text
1) "page1"
2) "page2"
3) "page3"
```

### Keep Last N with RPUSH

If items are appended to the tail (RPUSH), keep the most recent N with a negative stop index.

```redis
RPUSH events "e1" "e2" "e3" "e4" "e5" "e6"
LTRIM events -3 -1
LRANGE events 0 -1
```

```text
1) "e4"
2) "e5"
3) "e6"
```

### Out-of-Range Start Clamps the List

If `start` is greater than the end of the list, the key is deleted.

```redis
RPUSH mylist "x" "y" "z"
LTRIM mylist 10 20
EXISTS mylist
```

```text
(integer) 0
```

### Atomic Capped List Pattern

Combine LPUSH and LTRIM in a pipeline to maintain a fixed-length list atomically.

```redis
LPUSH activityfeed "action:login"
LTRIM activityfeed 0 99
```

This keeps the 100 most recent activities in a single round trip when pipelined.

## Use Cases

### Capped Log Buffer

Maintain a rolling window of the last 1000 log lines.

```redis
RPUSH applog "ERROR: connection refused"
LTRIM applog -1000 -1
```

Each new log entry is appended, and the list is trimmed to the last 1000 entries.

### Recent Items Feed

Keep a user's recently viewed items list to a fixed size.

```redis
LPUSH user:42:recent "item:789"
LTRIM user:42:recent 0 19
```

This ensures the list never exceeds 20 items.

### Sliding Window Rate Limiter

Store timestamps of recent requests and trim to a time window.

```redis
RPUSH ratelimit:user:42 "1711900000"
LTRIM ratelimit:user:42 -100 -1
```

Combined with LLEN, this gives the request count within the window.

### Cleanup After Bulk Insert

After loading a batch of data, trim to keep only valid range entries.

```redis
RPUSH pipeline:results "res1" "res2" "res3" "res4" "res5"
LTRIM pipeline:results 0 2
LRANGE pipeline:results 0 -1
```

```text
1) "res1"
2) "res2"
3) "res3"
```

## Performance Considerations

- LTRIM is O(N) where N is the number of elements removed.
- When trimming to keep a small tail of a large list, the operation touches many elements. Consider using a deque-style approach with LPOP/RPOP if single-element removal at edges is sufficient.
- LTRIM is safe to run repeatedly because it is idempotent when the list is already within range.

## Summary

`LTRIM` is an efficient way to enforce size limits on Redis lists by pruning them to a specified index range. It is the foundation of the capped-list pattern widely used for activity feeds, log buffers, and sliding-window rate limiters. Pair it with LPUSH or RPUSH in a pipeline to maintain a fixed-length list with every insert.
