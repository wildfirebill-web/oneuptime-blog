# How Partial Resynchronization Works in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Replication, PSYNC, Partial Resync, Internals

Description: A deep dive into how Redis partial resynchronization (PSYNC) works, including replication IDs, offsets, and when full vs partial sync occurs.

---

## Why Partial Resynchronization Matters

When a replica reconnects to a primary after a network interruption, it needs to re-sync. Without partial resynchronization (introduced in Redis 2.8), every reconnect triggered a full synchronization - a costly operation where the primary creates a full RDB snapshot and sends it to the replica. For large datasets and unreliable networks, this was very expensive.

Partial resynchronization allows a replica to receive only the commands it missed during the disconnection, dramatically reducing network traffic and recovery time.

## Key Concepts

### Replication ID

Every Redis primary has a **replication ID** - a pseudo-random string generated at startup or when a replica becomes a primary. It identifies a specific dataset lineage:

```bash
redis-cli INFO replication | grep master_replid
```

Output:

```text
master_replid:8f4b3c9a7e5d2a1b0c3d4e5f6a7b8c9d0e1f2a3b
master_replid2:0000000000000000000000000000000000000000
```

`master_replid2` is the previous replication ID, kept for partial sync after a failover.

### Replication Offset

Both the primary and each replica maintain a **replication offset** - a byte counter tracking how many bytes of the replication stream have been processed:

```bash
redis-cli INFO replication | grep -E "master_repl_offset|slave_repl_offset"
```

The offset increases monotonically as commands are replicated.

### Replication Backlog

The primary maintains a circular **replication backlog** buffer that stores recent replication stream data. The backlog enables partial resynchronization by allowing replicas to request commands starting at a specific offset:

```text
# Default backlog size: 1 MB
repl-backlog-size 1mb
repl-backlog-ttl 3600
```

## The PSYNC Command

When a replica reconnects, it sends `PSYNC <replid> <offset>` to the primary:

```text
> PSYNC 8f4b3c9a7e5d2a1b0c3d4e5f6a7b8c9d0e1f2a3b 5000
```

The primary evaluates the request and responds with one of:

- `+CONTINUE <replid>` - partial sync is possible, stream the missing commands
- `+FULLRESYNC <replid> <offset>` - full sync required

## When Partial Sync Succeeds

Partial sync is possible when:

1. The replica's replication ID matches the primary's `master_replid` or `master_replid2`
2. The offset the replica sends is within the current backlog range

```text
backlog_first_offset <= replica_offset <= master_repl_offset
```

Example: if the backlog holds offsets 4000 to 5000, and the replica was at offset 4800, the primary sends bytes 4800 to 5000 (and continues streaming new commands).

## When Full Sync Is Required

A full sync (`FULLRESYNC`) is triggered when:

- The replica is connecting for the first time
- The replica's replication ID does not match (e.g., the replica connected to a different primary)
- The replica's offset is older than the backlog covers (the needed data was overwritten)
- The replica was using `REPLICAOF NO ONE` and is reconnecting

Full sync process:

1. Primary calls `BGSAVE` to create an RDB snapshot (or uses a disk-less transfer)
2. Snapshot is sent to the replica
3. Replica flushes its dataset and loads the RDB
4. Replication stream resumes from the snapshot's offset

## Monitoring Sync Type

The Redis log shows which type of sync occurred:

```text
# Partial sync triggered
* Partial resynchronization request from 192.168.1.11:52345 accepted. Sending 204800 bytes of backlog starting from offset 4800.

# Full sync triggered
* Partial resynchronization not possible (no cached master). Full resync triggered.
```

You can also see sync counts in `INFO stats`:

```bash
redis-cli INFO stats | grep -E "sync_full|sync_partial"
```

Output:

```text
sync_full:2
sync_partial_ok:15
sync_partial_err:1
```

`sync_partial_err` means the partial sync failed (offset not in backlog) and fell back to full sync.

## Tuning the Backlog to Avoid Full Syncs

If `sync_partial_err` is increasing, your backlog is too small. Increase it:

```bash
redis-cli CONFIG SET repl-backlog-size 50mb
```

Rule of thumb: set the backlog to cover at least 60 seconds of your peak write throughput. If you write 10 MB/s, set a 600 MB backlog:

```text
repl-backlog-size 600mb
```

## Partial Sync After Failover (Redis 4.0+)

Redis 4.0 added support for partial sync after failover. When a replica is promoted to primary, it keeps both the old replication ID (now in `master_replid2`) and generates a new one. Other replicas can still partial-sync against the new primary using the old ID:

```text
master_replid:newid123...
master_replid2:oldid456...  <- old primary's ID
second_repl_offset:5001     <- offset where the old ID's data ends
```

## Diskless Replication

For large datasets or ephemeral storage, use diskless replication to transfer the RDB directly over the network without writing to disk on the primary:

```text
repl-diskless-sync yes
repl-diskless-sync-delay 5
```

The delay gives multiple replicas time to connect before the transfer starts, sharing one RDB stream.

## Summary

Redis partial resynchronization uses replication IDs and byte offsets to efficiently catch up replicas after a brief disconnect, avoiding expensive full RDB transfers. The primary's replication backlog stores recent commands to enable this. To maximize partial sync success, configure a large enough backlog to cover your worst-case network outage duration times your write throughput. Redis 4.0+ extended partial sync to work after failovers, making recovery even more efficient.
