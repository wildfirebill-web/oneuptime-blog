# How to Use COMMAND GETKEYS in Redis to Extract Keys from Commands

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, COMMAND GETKEYS, Cluster, Key Extraction, Introspection

Description: Learn how to use COMMAND GETKEYS in Redis to extract the key names from a command string, essential for Redis Cluster routing and security analysis.

---

## What Is COMMAND GETKEYS?

`COMMAND GETKEYS` takes a full Redis command and its arguments, then returns the list of key names that command would operate on. It is primarily useful in Redis Cluster environments where commands must be routed to the correct shard based on key hash slots.

## Basic Syntax

```text
COMMAND GETKEYS command [arg [arg ...]]
```

## Basic Examples

```bash
redis-cli COMMAND GETKEYS SET foo bar
```

Output:

```text
1) "foo"
```

```bash
redis-cli COMMAND GETKEYS MSET key1 val1 key2 val2
```

Output:

```text
1) "key1"
2) "key2"
```

```bash
redis-cli COMMAND GETKEYS ZADD leaderboard 100 "player1" 200 "player2"
```

Output:

```text
1) "leaderboard"
```

## Use Case: Cluster Routing

In Redis Cluster, each key belongs to a hash slot (0-16383). All keys in a multi-key command must map to the same slot. `COMMAND GETKEYS` lets you inspect which keys a command touches before sending it:

```bash
redis-cli COMMAND GETKEYS MGET user:1 user:2 user:3
```

Output:

```text
1) "user:1"
2) "user:2"
3) "user:3"
```

If `user:1` and `user:2` hash to different slots, the cluster will reject `MGET` with a `CROSSSLOT` error. Use hash tags like `{user}:1` to force them to the same slot.

## Complex Commands

For commands with variable key positions like `EVAL`:

```bash
redis-cli COMMAND GETKEYS EVAL "return redis.call('get',KEYS[1])" 1 mykey
```

Output:

```text
1) "mykey"
```

The `1` here is the numkeys argument telling Redis how many keys follow.

```bash
redis-cli COMMAND GETKEYS SORT mylist STORE destkey
```

Output:

```text
1) "mylist"
2) "destkey"
```

## Use Case: Security Auditing

In zero-trust or multi-tenant environments, you can use `COMMAND GETKEYS` to audit which keys a given command string would access before allowing it through a proxy:

```python
def get_command_keys(command_parts):
    return redis_client.execute_command('COMMAND', 'GETKEYS', *command_parts)
```

## Difference from COMMAND INFO

`COMMAND INFO` returns static metadata about where keys appear in a command template. `COMMAND GETKEYS` resolves the actual keys for a specific command invocation - accounting for variable-length arguments.

## Summary

`COMMAND GETKEYS` extracts the actual key names from a Redis command invocation. It is essential for Redis Cluster clients that need to route multi-key commands correctly and for security tools that need to audit key access patterns before executing commands.
