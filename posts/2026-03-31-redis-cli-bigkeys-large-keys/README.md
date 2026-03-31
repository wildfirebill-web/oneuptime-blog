# How to Use Redis CLI --bigkeys for Finding Large Keys

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, CLI, Memory, Performance, Key Management

Description: Learn how to use redis-cli --bigkeys to scan your keyspace and identify the largest keys by data type, helping you find memory inefficiencies and hotspots.

---

Large keys in Redis consume excessive memory, cause slow operations when accessed, and can block the server during deletion. The `--bigkeys` flag in redis-cli scans the full keyspace using `SCAN` and reports the largest key found for each data type.

## Running --bigkeys

```bash
redis-cli --bigkeys
```

Output example:

```text
# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type. You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (slower but more production-safe).

[00.00%] Biggest string found so far '"user:profile:1234"' with 8192 bytes
[25.10%] Biggest hash   found so far '"product:catalog:9"' with 4320 fields
[50.20%] Biggest list   found so far '"events:queue:global"' with 120000 items
[75.30%] Biggest zset   found so far '"leaderboard:global"' with 85000 members

-------- summary -------

Sampled 2000000 keys in the keyspace!
Total key length in bytes is 45231000 (avg len 22.61)

Biggest   string found '"user:profile:1234"' has 8192 bytes
Biggest     hash found '"product:catalog:9"' has 4320 fields
Biggest     list found '"events:queue:global"' has 120000 items
Biggest    zset found '"leaderboard:global"' has 85000 members

32410 strings with 3.14 bytes per key (avg)
14200 hashes with 42 fields per key (avg)
5000 lists with 230 items per key (avg)
1000 zsets with 500 members per key (avg)
```

## Slowing Down the Scan

On a production server, add `-i` to introduce a sleep interval:

```bash
redis-cli --bigkeys -i 0.1
```

This sleeps 0.1 seconds between every 100 `SCAN` iterations, reducing CPU impact.

## Connecting to a Remote Instance

```bash
redis-cli -h redis.example.com -p 6379 -a password --bigkeys -i 0.05
```

## What to Do with Large Keys

### Large Strings (>100KB)

Consider splitting the value across multiple keys or storing large objects in blob storage (S3) with only a reference in Redis.

### Large Hashes (many fields)

Break large hashes into smaller ones by sharding on a field prefix:

```bash
# Instead of one hash with 10,000 fields:
HSET product:catalog field1 val1 field2 val2 ...

# Use partitioned hashes:
HSET product:catalog:0 field1 val1 ...
HSET product:catalog:1 field5001 val5001 ...
```

### Large Lists (many items)

If used as a queue, ensure consumers are keeping up. Use a capped list:

```bash
LPUSH events:queue:global "new-event"
LTRIM events:queue:global 0 9999
```

### Large Sorted Sets

Consider partitioning by score range or archiving old members:

```bash
# Remove members with score older than 30 days
ZREMRANGEBYSCORE leaderboard:global -inf 1697000000
```

## Finding the Actual Memory Size

After identifying a large key, measure its exact memory footprint:

```bash
redis-cli MEMORY USAGE events:queue:global
# (integer) 18432040
```

## Summary

`redis-cli --bigkeys` is an essential tool for memory audits, scanning the full keyspace to report the largest key for each data type. Use `-i` to throttle the scan on production and follow up with `MEMORY USAGE` to measure exact bytes per key.
