# How to Use Read Concern "available" in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Read Concern, Sharding, Performance, Consistency

Description: Learn how MongoDB's read concern "available" provides the lowest-latency reads from sharded clusters by skipping ownership checks, and understand its consistency trade-offs.

---

## What Is Read Concern "available"?

Read concern `"available"` returns data from the queried node as quickly as possible without performing any ownership verification. It is the lowest-latency read concern available in MongoDB and is primarily designed for use with sharded clusters where chunk migration may be in progress.

`"available"` is similar to `"local"` for replica sets, but differs significantly for sharded clusters.

## available vs. local on Sharded Clusters

On sharded clusters, chunk migrations can cause data to exist on two shards simultaneously during the migration window. Read concern `"local"` performs a check to determine which shard owns the chunk and returns only owned data. Read concern `"available"` skips this check:

```text
Read Concern     Checks chunk ownership   Latency   Risk
local            Yes                      Low       None
available        No                       Lowest    Orphan reads
majority         Yes (committed)          Moderate  None
```

The `"available"` risk is returning "orphan documents" - documents that have been migrated away from this shard but not yet cleaned up. These documents would not exist in any correct view of the dataset.

## When to Use "available"

Use `"available"` only when:

- You are querying a sharded cluster
- You can tolerate orphan document reads
- Absolute lowest latency is critical (e.g., real-time analytics where exact counts are not required)
- You understand that results may include documents that belong to another shard

For replica sets, `"available"` behaves identically to `"local"`.

## Setting Read Concern to "available"

```javascript
db.events.find({ timestamp: { $gte: ISODate("2026-01-01") } })
  .readConcern("available")
```

Or in the driver:

```javascript
const results = await db.collection("events").find(
  { timestamp: { $gte: new Date("2026-01-01") } },
  { readConcern: { level: "available" } }
).toArray();
```

## What Are Orphan Documents?

During chunk migration, MongoDB moves data from one shard to another in ranges. While a range is being moved, the source shard may still hold the data. If a read with `"available"` hits the source shard during this window, it may return documents that have already logically moved:

```text
Shard A holds chunks [1-100]
Migration starts: moving [51-100] to Shard B

With available: Shard A may still return docs 51-100
With local:     Shard A skips docs 51-100 (acknowledges migration)
```

## Appropriate Use Case: Approximate Analytics

For a dashboard that counts approximate event volumes per region:

```javascript
const approxCount = await db.collection("events").countDocuments(
  { region: "us-east" },
  { readConcern: { level: "available" } }
);

// Acceptable: the count may be slightly off during chunk migrations
console.log(`Approximate US East events: ${approxCount}`);
```

This reduces p99 latency at the cost of occasional slightly incorrect counts.

## Comparison Across All Read Concerns

```text
Level            Latency    Rollback Safe   Orphan Reads   Transaction
available        Lowest     No              Possible       No
local            Low        No              No             No
majority         Moderate   Yes             No             No
linearizable     Highest    Yes             No             No (single doc)
snapshot         Moderate   Yes             No             Yes (required)
```

## Production Recommendations

- Do not use `"available"` as the default read concern for most applications
- Use `"local"` instead of `"available"` on non-sharded deployments (they are equivalent)
- If using `"available"` on a sharded cluster, run periodic orphan cleanup with the `cleanupOrphans` command
- Document clearly in code comments why `"available"` is being used, to avoid confusion

## Summary

Read concern `"available"` provides the absolute lowest read latency on sharded clusters by skipping chunk ownership checks. This makes it possible to read orphan documents during chunk migrations. It is appropriate only for workloads that tolerate approximate results - such as analytics dashboards - and should be avoided in applications where data correctness is critical. For standard low-latency reads, `"local"` is the safer equivalent.
