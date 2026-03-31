# How to Use PSYNC in Redis for Partial Resynchronization

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Replication, Command

Description: Learn how the Redis PSYNC command works to enable partial resynchronization between a master and replica, reducing full resync overhead.

---

`PSYNC` is an internal Redis command used during replication to request either a partial or full resynchronization from a master. When a replica reconnects after a brief disconnect, `PSYNC` allows it to catch up with only the missed commands rather than transferring the entire dataset again - a process called partial resynchronization.

> Note: `PSYNC` is not intended to be called directly by application code. It is invoked internally by replica nodes. Understanding how it works is valuable for diagnosing replication issues.

## Syntax

```text
PSYNC <replicationid> <offset>
```

- `replicationid`: The replication ID the replica last saw from the master
- `offset`: The byte offset up to which the replica has received data

## How Partial Resynchronization Works

When a replica connects (or reconnects) to a master, it sends `PSYNC` with its last known replication ID and offset. The master checks whether the requested offset is still within its replication backlog.

```text
Replica -> Master: PSYNC <repid> <offset>

If offset is in backlog:
  Master -> Replica: +CONTINUE <repid>
  Master sends only the missed commands

If offset is NOT in backlog:
  Master -> Replica: +FULLRESYNC <repid> <offset>
  Master sends a full RDB snapshot
```

## Triggering a Full Resync Intentionally

To force a full resync, pass `?` as the replication ID and `-1` as the offset:

```bash
PSYNC ? -1
```

This is what a brand-new replica sends when it connects to a master for the first time.

## Replication Backlog

The ability to perform a partial resync depends on the replication backlog. Configure its size in `redis.conf`:

```text
repl-backlog-size 10mb
repl-backlog-ttl 3600
```

If the replica was offline for too long and the missed commands no longer fit in the backlog, Redis falls back to a full resync automatically.

## Monitoring PSYNC Activity

Use `INFO replication` to observe partial vs full sync events:

```bash
redis-cli INFO replication
```

Look for these fields:

```text
sync_full:2
sync_partial_ok:15
sync_partial_err:1
```

- `sync_full`: Number of full RDB syncs performed
- `sync_partial_ok`: Number of successful partial syncs
- `sync_partial_err`: Number of partial sync attempts that had to fall back to full sync

## Diagnosing PSYNC Failures

When `sync_partial_err` is climbing, the backlog may be too small. Increase `repl-backlog-size` to reduce full resyncs and lower replication bandwidth:

```bash
redis-cli CONFIG SET repl-backlog-size 50mb
```

Verify the change:

```bash
redis-cli CONFIG GET repl-backlog-size
```

## Python: Checking Replication Health

```python
import redis

r = redis.Redis(host="127.0.0.1", port=6379)
info = r.info("replication")

print(f"Full syncs: {info['sync_full']}")
print(f"Partial syncs OK: {info['sync_partial_ok']}")
print(f"Partial syncs failed: {info['sync_partial_err']}")

if info["sync_partial_err"] > 0:
    print("Warning: some partial syncs fell back to full resync")
    print("Consider increasing repl-backlog-size")
```

## Summary

`PSYNC` is the protocol-level mechanism that enables Redis replicas to resume replication from where they left off after a brief disconnection. While application developers rarely call it directly, understanding PSYNC and the replication backlog is essential for tuning Redis replication efficiency, reducing bandwidth usage, and diagnosing sync failures in production environments.
