# How Redis Handles Stream Trimming During Consumer Reads

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Stream, Consumer

Description: Understand how Redis handles XADD MAXLEN trimming while consumers are actively reading, what happens to pending messages, and how to trim safely.

---

Redis Streams grow indefinitely unless you trim them. Trimming during active consumer reads requires careful handling to avoid losing pending, unacknowledged messages. This post explains how trimming works and how to do it safely.

## How Stream Trimming Works

Trimming removes the oldest entries from a stream. You can trim at write time using `XADD MAXLEN` or explicitly using `XTRIM`:

```bash
# Trim to maximum 1000 entries on each write
redis-cli XADD mystream MAXLEN 1000 '*' field value

# Trim explicitly
redis-cli XTRIM mystream MAXLEN 1000
```

Redis trims from the beginning of the stream (oldest entries first).

## The Risk: Trimming Pending Messages

Consumer groups track pending messages - messages that were delivered to a consumer but not yet acknowledged. If you trim a stream and remove entries that are still in the Pending Entries List (PEL), those messages become orphaned.

Check pending messages before trimming:

```bash
redis-cli XPENDING mystream mygroup - + 100
```

If any message IDs shown in XPENDING are older than what XTRIM would keep, trimming will orphan them.

## Safe Trimming Strategy

Always trim to a point older than the oldest unacknowledged message:

```bash
# Find the oldest pending message ID
redis-cli XPENDING mystream mygroup - + 1

# Trim to keep entries older than that ID
redis-cli XTRIM mystream MINID <oldest-pending-id>
```

`MINID` trims all entries with IDs older than the specified ID, which is safer than `MAXLEN` when you have active consumers.

## Approximate Trimming

Exact trimming requires Redis to traverse the internal data structure and can be slow for large streams. Use the `~` modifier for approximate trimming, which is much faster:

```bash
redis-cli XADD mystream MAXLEN ~ 1000 '*' field value
redis-cli XTRIM mystream MAXLEN ~ 1000
```

The `~` allows Redis to trim in full radix-tree nodes, which is significantly faster on large streams.

## What Happens to Orphaned Pending Messages

If a trimmed message is still in the PEL, the consumer group references a message ID that no longer exists in the stream. When a consumer tries to claim or acknowledge it:

```bash
redis-cli XACK mystream mygroup <orphaned-id>
# Returns: 0 (nothing was acknowledged)
```

The acknowledgment silently fails. To clean up orphaned PEL entries, remove them manually:

```bash
redis-cli XDEL mystream <orphaned-id>
```

## Monitoring Stream Size

Keep an eye on stream length before and after trimming:

```bash
redis-cli XLEN mystream
redis-cli XINFO STREAM mystream
redis-cli XINFO GROUPS mystream
```

## Summary

Redis stream trimming during active consumer reads can orphan pending unacknowledged messages. Use `XTRIM MINID` based on the oldest pending message ID for safe trimming, and use the `~` approximate modifier for performance. Always check `XPENDING` before trimming to avoid data loss in consumer groups.
