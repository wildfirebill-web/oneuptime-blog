# How to Use Read Preference 'secondary' in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Read Preference, Replica Set, Secondary, Scalability

Description: Learn how MongoDB's read preference 'secondary' routes all reads to secondary members, reducing primary load and enabling read scalability for analytics workloads.

---

## What Is Read Preference "secondary"?

Read preference `"secondary"` routes all read operations to secondary members of a replica set. The primary is never used for reads. If no secondary is available, the operation fails - it does not fall back to the primary.

This mode is designed for workloads that explicitly want to offload reads from the primary and can tolerate some replication lag.

## Setting Read Preference to secondary

In `mongosh`:

```javascript
db.reports.find({ month: "2026-03" }).readPref("secondary")
```

In the connection string:

```text
mongodb://mongo1:27017,mongo2:27017,mongo3:27017/myapp?readPreference=secondary&replicaSet=rs0
```

In the Node.js driver for a specific collection:

```javascript
const { ReadPreference } = require("mongodb");

const reportsCollection = db
  .collection("reports")
  .withReadPreference(ReadPreference.SECONDARY);

const results = await reportsCollection.find({ month: "2026-03" }).toArray();
```

## The Staleness Trade-Off

Secondaries replicate asynchronously. A secondary may lag behind the primary by milliseconds to seconds depending on write volume and network conditions:

```text
Primary writes at T=100ms
Secondary 1 applies at T=105ms (5ms lag - healthy)
Secondary 2 applies at T=300ms (200ms lag - check the network)

readPref:"secondary" may return either secondary's state
```

For read-heavy analytics queries where a few seconds of lag is acceptable, this trade-off is worthwhile.

## When to Use "secondary"

`"secondary"` is appropriate for:

- **Reporting and analytics** - nightly or hourly reports that run against historical data
- **Background jobs** - data exports, ETL pipelines, audit jobs
- **Search indexing** - building search index from MongoDB data
- **Development/testing** - reading replica data without impacting production primary
- **Reducing primary CPU** - offloading expensive aggregation queries

## When NOT to Use "secondary"

Avoid `"secondary"` when:

- Reads must see the result of recent writes (use `"primary"`)
- You need a consistent view of rapidly changing data
- The application will fail if the result is stale
- Write and read operations are interleaved (shopping cart, inventory deduction)

## Limiting Staleness With maxStalenessSeconds

Prevent reading from a secondary that is severely lagging:

```javascript
const { ReadPreference } = require("mongodb");

const readPref = new ReadPreference("secondary", null, {
  maxStalenessSeconds: 120
});

const results = await collection.find({}).withReadPreference(readPref).toArray();
```

This skips any secondary whose replication lag exceeds 120 seconds. If all secondaries exceed the threshold, the driver throws an error rather than returning stale data.

The minimum valid value is 90 seconds (enforced by the driver).

## Load Distribution

In a 3-node replica set (1 primary, 2 secondaries), `"secondary"` distributes reads across the 2 secondaries using a round-robin or latency-based selection:

```text
Read 1  ->  Secondary 1
Read 2  ->  Secondary 2
Read 3  ->  Secondary 1
Read 4  ->  Secondary 2
```

This effectively doubles your read throughput capacity for secondary reads.

## Separate Analytics Connection

A common pattern is to create a separate MongoDB client for analytics with `"secondary"` preference:

```javascript
const primaryClient = new MongoClient(uri, {
  readPreference: "primary"
});

const analyticsClient = new MongoClient(uri, {
  readPreference: "secondary",
  maxStalenessSeconds: 300
});

// Use primaryClient for user-facing reads and writes
// Use analyticsClient for reporting queries
```

## Failure Mode

If both secondaries are unavailable:

```text
Secondary 1 down
Secondary 2 down
readPref:"secondary" -> read fails (does NOT fall back to primary)
```

For resilience with fallback, use `"secondaryPreferred"` instead.

## Tag Sets for Dedicated Reporting Secondaries

Tag a secondary specifically for reporting and target it:

```javascript
// In replica set config:
// { "tags": { "purpose": "reporting", "region": "us-east" } }

const readPref = new ReadPreference("secondary", [
  { purpose: "reporting" }
]);
```

## Summary

Read preference `"secondary"` sends all reads to secondary replica set members, relieving the primary of read traffic. It is well-suited for analytics, reporting, and background jobs where some replication lag is acceptable. Use `maxStalenessSeconds` to bound how stale a secondary can be, and consider `"secondaryPreferred"` if you need fallback to the primary when no secondaries are available.
