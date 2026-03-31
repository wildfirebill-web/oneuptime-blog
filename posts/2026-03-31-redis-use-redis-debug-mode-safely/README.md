# How to Use Redis Debug Mode Safely

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Debugging, Operations

Description: A practical guide to using Redis DEBUG commands safely in development and production - what each subcommand does and when to use it.

---

Redis ships with a built-in `DEBUG` command that exposes low-level introspection and testing utilities. While extremely useful for diagnosing issues, many DEBUG subcommands are destructive or affect performance. This guide explains how to use them safely.

## What the DEBUG Command Is

The `DEBUG` command is not meant for regular application use. It is a tool for Redis developers and operators. Always be careful running it in production.

```bash
redis-cli DEBUG help
```

## Safe Subcommands

### DEBUG SLEEP

Causes Redis to sleep for a given number of seconds. Useful for simulating a slow Redis in integration tests:

```bash
redis-cli DEBUG SLEEP 5
```

Do not run this in production as it blocks the entire server.

### DEBUG JMAP

Prints a count of objects by type in memory. Safe to run but may take time on large datasets:

```bash
redis-cli DEBUG JMAP
```

### DEBUG RELOAD

Forces Redis to save the RDB file and reload it. Useful for testing persistence without restarting:

```bash
redis-cli DEBUG RELOAD
```

### DEBUG LOADAOF

Triggers an immediate AOF load. This is used for testing AOF recovery:

```bash
redis-cli DEBUG LOADAOF
```

## Risky Subcommands

### DEBUG SEGFAULT

Immediately crashes the Redis server. Only use in isolated test environments:

```bash
# NEVER run in production
redis-cli DEBUG SEGFAULT
```

### DEBUG FLUSHALL async

While not a DEBUG subcommand, calling `FLUSHALL` without a backup is destructive. Always confirm before running.

## Restricting DEBUG in Production

To prevent accidental use, rename or disable the DEBUG command in `redis.conf`:

```bash
# redis.conf
rename-command DEBUG ""
```

Or rename it to an obscure string:

```bash
rename-command DEBUG "a7f3b2c9d1e4f5a6"
```

## Using DEBUG OBJECT

This subcommand is safe and useful for inspecting encoding and memory of a specific key:

```bash
redis-cli DEBUG OBJECT mykey
```

Output example:

```text
Value at:0x7f3a2001d2f0 refcount:1 encoding:ziplist serializedlength:18 lru:14563281 lru_seconds_idle:2 type:list
```

This tells you how Redis is storing the key internally, which helps with performance tuning.

## Protecting Your Cluster

If you're using Redis Cluster, avoid running DEBUG RELOAD or DEBUG SLEEP on primary nodes. Run these only on replicas or in maintenance windows.

## Summary

Redis DEBUG mode provides powerful introspection and testing capabilities, but several subcommands can crash or block the server. Always restrict DEBUG access in production via `rename-command`, and limit usage to non-destructive subcommands like `DEBUG OBJECT` or `DEBUG RELOAD` when needed in live environments.
