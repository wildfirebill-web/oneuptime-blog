# What Is Redis Replication Offset

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Replication, Architecture

Description: Understand what the Redis replication offset is, how it tracks replication progress, how it is used for partial resync, and how to monitor replica lag.

---

The replication offset is a counter that tracks how much of the replication stream has been processed. It is central to how Redis determines whether a replica needs a full resync or just a partial catch-up after a disconnect. Understanding it helps you diagnose replication lag and recovery scenarios.

## What Is the Replication Offset?

Every write command that executes on the primary is appended to the replication buffer and assigned an offset - a byte position in the stream of replication data. The primary maintains `master_repl_offset`, and each replica maintains its own `slave_repl_offset`.

Check these values:

```bash
redis-cli INFO replication | grep -E "master_repl_offset|slave_repl_offset"
```

## How Offset Works During Replication

1. Primary executes a write command and advances `master_repl_offset`
2. The command is appended to the replication backlog
3. The command is sent to all connected replicas
4. Each replica processes the command and advances its own `slave_repl_offset`
5. Replication lag = `master_repl_offset` - `slave_repl_offset`

## Monitoring Replication Lag

The lag between primary and replica in bytes:

```bash
redis-cli INFO replication | grep -E "master_repl_offset|slave_repl_offset|lag"
```

Convert bytes to approximate command count for context. Even a small byte offset can represent many small SET commands.

## Partial Resync vs Full Resync

When a replica reconnects after a disconnect, it sends its last known offset to the primary. The primary checks whether that offset is still in the replication backlog:

```bash
redis-cli CONFIG GET repl-backlog-size
```

If the replica's offset is within the backlog window, Redis performs a **partial resync** - sending only the missing commands. This is fast and avoids full RDB transfer.

If the offset has fallen out of the backlog (backlog too small or replica was offline too long), Redis performs a **full resync** - generating a new RDB snapshot and sending it to the replica. This is slow and resource-intensive.

## The Replication Backlog

The backlog is a circular buffer on the primary that stores recent replication stream data. When it fills up, old data is overwritten:

```bash
redis-cli CONFIG GET repl-backlog-size
redis-cli INFO replication | grep "repl_backlog_active"
```

Increase the backlog for replicas that may be offline for extended periods:

```bash
redis-cli CONFIG SET repl-backlog-size 512mb
```

## Run ID and Offsets

Each Redis instance has a `run_id`. When a primary restarts, its run_id changes, and replicas must perform a full resync even if their offset is still valid, because the offset refers to a previous primary's stream:

```bash
redis-cli INFO server | grep run_id
redis-cli INFO replication | grep master_replid
```

## Summary

The Redis replication offset is a byte counter that tracks how much of the replication command stream has been processed. Replicas use their saved offset to request partial resync after reconnecting. If the offset falls outside the replication backlog, a full resync is required. Keeping the backlog large enough for your replica disconnect window prevents costly full resyncs.
