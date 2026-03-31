# How to Use TOUCH in Redis to Update Key Access Time

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, TOUCH, LRU, Key Management, Eviction

Description: Learn how to use the Redis TOUCH command to update the last access time of one or more keys without reading their values.

---

The `TOUCH` command in Redis updates the last access time of one or more keys. This is useful when you want to prevent a key from being evicted by an LRU (Least Recently Used) eviction policy, without actually reading or modifying the key's value.

## Syntax

```text
TOUCH key [key ...]
```

The command accepts one or more key names and returns the number of keys that were "touched" - meaning they exist and had their access time updated.

## Basic Usage

```bash
# Touch a single key
TOUCH session:user:1001

# Touch multiple keys at once
TOUCH session:user:1001 session:user:1002 session:user:1003
```

The return value is the count of existing keys among those specified. Non-existent keys are silently ignored.

## Why TOUCH Matters with LRU Eviction

When Redis is configured with an LRU-based eviction policy such as `allkeys-lru` or `volatile-lru`, it evicts keys that have not been accessed recently. If you need to keep a key alive without reading it - perhaps for cache warming or marking keys as still-in-use - `TOUCH` is the right tool.

```bash
# Configure maxmemory and eviction policy
CONFIG SET maxmemory 256mb
CONFIG SET maxmemory-policy allkeys-lru

# Prevent key eviction by refreshing its access time
TOUCH precomputed:report:2024-q4
```

## Checking Access Time with OBJECT IDLETIME

You can verify that `TOUCH` works by observing the idle time drop after calling it:

```bash
SET mykey "hello"
# Wait a few seconds...
OBJECT IDLETIME mykey
# Returns: 5

TOUCH mykey
OBJECT IDLETIME mykey
# Returns: 0
```

## Touching Multiple Keys in a Pipeline

For high-throughput scenarios, pipeline `TOUCH` commands to reduce round-trip latency:

```bash
redis-cli --pipe <<'EOF'
TOUCH session:user:1001
TOUCH session:user:1002
TOUCH session:user:1003
EOF
```

## Practical Example - Keeping Active Sessions Alive

In a session management system, you might want to refresh access times for all sessions that have been active recently without loading their data:

```bash
# Refresh a batch of active session keys
TOUCH \
  session:abc123 \
  session:def456 \
  session:ghi789
```

This keeps these keys "warm" in the LRU cache so they are less likely to be evicted when memory pressure increases.

## TOUCH vs GET for Access Time Updates

Both `GET` and `TOUCH` update a key's LRU clock. The difference is that `GET` returns the key's value (causing a network transfer), while `TOUCH` only updates metadata. For large values, `TOUCH` is significantly more efficient when you only need to refresh the eviction timer.

## Summary

`TOUCH` is a lightweight Redis command that updates the last access time for one or more keys without transferring their data. It is especially useful in LRU eviction setups where you want to protect frequently-needed but rarely-read keys from being evicted. Combined with `OBJECT IDLETIME`, it gives you fine-grained control over Redis memory management.
