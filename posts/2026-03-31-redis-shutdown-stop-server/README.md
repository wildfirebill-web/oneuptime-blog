# How to Use SHUTDOWN in Redis to Stop the Server Safely

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, SHUTDOWN, Server Management, Persistence, Operations

Description: Learn how to use the SHUTDOWN command in Redis to stop the server safely, with options to save data, skip saving, or force immediate termination.

---

## What Is the SHUTDOWN Command?

`SHUTDOWN` stops the Redis server process. It can optionally trigger a final data save before exiting, ensuring no data is lost. This is the recommended way to stop Redis cleanly rather than sending `SIGKILL`.

## Basic Syntax

```text
SHUTDOWN [NOSAVE | SAVE] [NOW] [FORCE] [ABORT]
```

All options are available since Redis 7.0. In older versions, only `NOSAVE` and `SAVE` are supported.

## Common Usage

### Graceful Shutdown (Save First)

```bash
redis-cli SHUTDOWN SAVE
```

Redis performs a final `BGSAVE`, waits for it to complete, then exits. Safe for RDB persistence.

### Shutdown Without Saving

```bash
redis-cli SHUTDOWN NOSAVE
```

Redis exits immediately without saving. Useful when you intentionally want to discard changes.

### Default SHUTDOWN Behavior

```bash
redis-cli SHUTDOWN
```

Without a modifier, Redis saves if persistence is configured (either RDB or AOF is enabled). If neither is enabled, it exits without saving.

## Advanced Options (Redis 7.0+)

### Immediate Shutdown

```bash
redis-cli SHUTDOWN NOW
```

Exits without waiting for background saves to complete.

### Force Shutdown

```bash
redis-cli SHUTDOWN FORCE
```

Forces shutdown even if there are connected replicas. Normally Redis waits for replicas to catch up.

### Abort Pending Shutdown

```bash
redis-cli SHUTDOWN ABORT
```

Cancels a pending `SHUTDOWN` that is waiting for replicas or saves to complete.

## What Happens on SHUTDOWN SAVE

1. Redis calls `BGSAVE` to write an RDB snapshot
2. If AOF is enabled, the AOF buffer is flushed to disk
3. All connected clients receive connection closed errors
4. The server process exits

## SHUTDOWN in Systemd

When Redis runs as a systemd service, systemd typically sends `SIGTERM` on `systemctl stop redis`. Redis handles `SIGTERM` as a graceful shutdown (equivalent to `SHUTDOWN SAVE`).

Check the service configuration:

```bash
sudo systemctl status redis
```

The `ExecStop` directive may call `redis-cli SHUTDOWN` explicitly.

## Who Can Run SHUTDOWN?

By default, only authenticated users with admin privileges can run `SHUTDOWN`. In Redis ACL:

```bash
redis-cli ACL SETUSER admin +shutdown
```

## Safety Considerations

- Never use `SHUTDOWN NOSAVE` on a production instance with valuable data unless you have an AOF backup
- In Redis Cluster, shutting down one node triggers failover - use `CLUSTER FAILOVER` first for zero-downtime operations
- Always verify persistence is working before relying on `SHUTDOWN SAVE`

## Summary

`SHUTDOWN` provides controlled, safe ways to stop a Redis server. Use `SHUTDOWN SAVE` for graceful shutdowns with data persistence, `SHUTDOWN NOSAVE` for intentional discard scenarios, and the newer `NOW`/`FORCE` modifiers when you need immediate termination in Redis 7.0+.
