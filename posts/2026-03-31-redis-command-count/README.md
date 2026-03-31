# How to Use COMMAND COUNT in Redis to Count Available Commands

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, COMMAND COUNT, Introspection, Version Detection, CLI

Description: Learn how to use COMMAND COUNT in Redis to get the total number of commands available on the server, useful for version detection and feature discovery.

---

## What Is COMMAND COUNT?

`COMMAND COUNT` returns the total number of Redis commands supported by the server. It is a fast, single-value query useful for comparing Redis versions, verifying module installations, and quick health checks.

## Basic Usage

```bash
redis-cli COMMAND COUNT
```

Example output:

```text
(integer) 246
```

The exact number varies by Redis version and installed modules.

## Why COMMAND COUNT Is Useful

### Version Fingerprinting

Different Redis versions support different numbers of commands. While not a reliable version check on its own, a sudden change in `COMMAND COUNT` after an upgrade indicates new commands were added:

```bash
# Before upgrade
redis-cli COMMAND COUNT
# (integer) 210

# After upgrade to Redis 7.x
redis-cli COMMAND COUNT
# (integer) 246
```

### Verifying Module Installation

Redis modules (like RedisJSON, RediSearch, RedisTimeSeries) add new commands. Checking `COMMAND COUNT` before and after loading a module confirms the module registered its commands:

```bash
# Before loading RedisJSON
redis-cli COMMAND COUNT
# (integer) 246

# After loading RedisJSON
redis-cli COMMAND COUNT
# (integer) 271
```

### Comparison Across Cluster Nodes

In a Redis Cluster, all nodes should support the same commands. If `COMMAND COUNT` differs between nodes, one may be running a different version or have a missing module:

```bash
for NODE in node1:6379 node2:6379 node3:6379; do
  echo "$NODE: $(redis-cli -h ${NODE%:*} -p ${NODE#*:} COMMAND COUNT)"
done
```

## Combining with COMMAND INFO

After checking the count, use `COMMAND INFO` to inspect specific commands:

```bash
redis-cli COMMAND INFO json.get
```

If the command exists, it returns metadata. If not, it returns `(nil)` - confirming a module is not loaded.

## Using in Scripts

```bash
#!/bin/bash
COUNT=$(redis-cli COMMAND COUNT)
if [ "$COUNT" -lt 200 ]; then
  echo "WARNING: Unexpectedly low command count: $COUNT"
  exit 1
fi
echo "Command count OK: $COUNT"
```

## Getting the Full Command List

If you need more than just the count:

```bash
redis-cli COMMAND | grep -E "^\s+[0-9]+\) \"" | head -20
```

Or use `COMMAND DOCS` for full documentation of all commands.

## Summary

`COMMAND COUNT` is a simple but useful introspection command that returns how many commands a Redis server supports. It helps detect version differences, verify module installations, and confirm consistent configuration across cluster nodes.
