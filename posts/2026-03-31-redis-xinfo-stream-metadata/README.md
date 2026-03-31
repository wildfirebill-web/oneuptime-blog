# How to Use XINFO STREAM in Redis to Inspect Stream Metadata

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Stream, XINFO, Monitoring, Observability

Description: Learn how to use XINFO STREAM to inspect Redis Stream metadata including length, first and last entry, consumer groups, and radix tree internals.

---

`XINFO STREAM` gives you a detailed snapshot of a Redis Stream's structure and contents. It is the first command to run when diagnosing stream health, verifying data was written correctly, or understanding the stream's size and age.

## How XINFO STREAM Works

`XINFO STREAM` reads the stream's internal metadata and returns it in a human-readable format. The full form (`FULL`) provides exhaustive detail including consumer group state, while the default form gives a concise summary.

## Syntax

```redis
XINFO STREAM key [FULL [COUNT count]]
```

- `key` - stream name
- `FULL` - return full detail including groups and PEL entries
- `COUNT count` - limit entries returned in FULL mode (default 10)

## Examples

### Basic Summary

```redis
XINFO STREAM mystream
```

Example output:

```text
 1) "length"
 2) (integer) 1250
 3) "radix-tree-keys"
 4) (integer) 3
 5) "radix-tree-nodes"
 6) (integer) 8
 7) "last-generated-id"
 8) "1711900500000-0"
 9) "max-deleted-entry-id"
10) "0-0"
11) "entries-added"
12) (integer) 1500
13) "recorded-first-entry-id"
14) "1711895000000-0"
15) "groups"
16) (integer) 2
17) "first-entry"
18) 1) "1711895000000-0"
   2) 1) "event"
      2) "user_login"
19) "last-entry"
20) 1) "1711900500000-0"
   2) 1) "event"
      2) "purchase"
```

Key fields explained:
- `length` - current number of entries
- `last-generated-id` - the highest ID ever assigned (monotonically increasing)
- `entries-added` - total entries ever added (including deleted ones)
- `groups` - number of consumer groups attached
- `first-entry` / `last-entry` - the oldest and newest messages in the stream

### Full Detail Mode

Get comprehensive information including consumer groups:

```redis
XINFO STREAM mystream FULL COUNT 5
```

This returns consumer group names, PEL sizes, consumer lists, and sample pending entries - useful for deep diagnostics.

### Checking Stream Age

Combine first and last entry IDs to calculate the stream's time span. The ID format is `<millisecond-timestamp>-<sequence>`:

```redis
XINFO STREAM mystream
```

Calculate age from the `first-entry` and `last-entry` timestamp components.

## Useful Fields for Monitoring

| Field | Purpose |
|---|---|
| `length` | Current entry count - monitor for growth |
| `groups` | Number of consumer groups - verify expected count |
| `last-generated-id` | Highest ever ID - useful for replication lag checks |
| `entries-added` | Total historical writes - useful for throughput metrics |
| `first-entry` | Oldest retained message - indicates retention window |

## Use Cases

- **Stream health dashboards** - regularly poll `XINFO STREAM` to monitor size and age
- **Deployment verification** - confirm data was written to the correct stream after a deploy
- **Capacity planning** - track `length` and `entries-added` over time to project growth
- **Consumer group audit** - quickly see how many groups are attached to a stream

## Summary

`XINFO STREAM` is the primary introspection command for Redis Streams. Use the default form for quick overviews and the `FULL` form for detailed debugging of consumer groups and pending entries. Monitoring the `length`, `first-entry`, and `last-entry` fields gives you immediate visibility into stream health and data retention.
