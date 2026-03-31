# How to Use Read Preference 'secondaryPreferred' in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Read Preference, Replica Set, Secondary, Availability

Description: Learn how MongoDB's read preference 'secondaryPreferred' routes reads to secondaries when available and falls back to the primary, balancing read scalability with availability.

---

## What Is Read Preference "secondaryPreferred"?

Read preference `"secondaryPreferred"` directs read operations to secondary members when they are available. If no secondary is reachable, it falls back to the primary. This is the most flexible read preference for distributing read load while maintaining high availability.

## Comparison with "secondary"

```text
Mode                 Secondary available   All secondaries down
secondary            Reads secondary       Fails (no fallback)
secondaryPreferred   Reads secondary       Falls back to primary
```

`"secondaryPreferred"` is the more resilient choice when read distribution is preferred but availability is also required.

## Setting Read Preference to secondaryPreferred

In `mongosh`:

```javascript
db.products.find({ inStock: true }).readPref("secondaryPreferred")
```

In the connection string:

```text
mongodb://mongo1:27017,mongo2:27017,mongo3:27017/myapp?readPreference=secondaryPreferred&replicaSet=rs0
```

In the Node.js driver:

```javascript
const { MongoClient, ReadPreference } = require("mongodb");

const client = new MongoClient(
  "mongodb://mongo1:27017,mongo2:27017,mongo3:27017/?replicaSet=rs0",
  { readPreference: ReadPreference.SECONDARY_PREFERRED }
);
```

## How MongoDB Selects the Secondary

When multiple secondaries are available, MongoDB uses a latency-based selection algorithm:

1. Measure round-trip time to each secondary
2. Identify the lowest-latency secondary
3. Include all secondaries within `localThresholdMS` of that latency (default: 15ms)
4. Select randomly from the eligible set

```javascript
// Adjust the latency threshold (default 15ms)
const client = new MongoClient(uri, {
  readPreference: "secondaryPreferred",
  localThresholdMS: 50  // include secondaries within 50ms of fastest
});
```

## Staleness Considerations

Secondaries replicate asynchronously, so `"secondaryPreferred"` reads may return slightly stale data. Limit how stale with `maxStalenessSeconds`:

```javascript
const { ReadPreference } = require("mongodb");

const readPref = new ReadPreference("secondaryPreferred", null, {
  maxStalenessSeconds: 90
});

const data = await collection.find({}).withReadPreference(readPref).toArray();
```

If all secondaries exceed the staleness threshold, the operation fails rather than returning stale data. The minimum valid value is 90 seconds.

## Practical Read Load Distribution

In a 3-node set, `"secondaryPreferred"` results in:

```text
Normal operation: primary handles writes + secondary 1 and 2 share reads
Secondary 1 down: primary handles writes + secondary 2 handles reads
Both secondaries down: primary handles writes AND reads (fallback)
```

This gives read scalability while maintaining full availability.

## Common Pattern: Application-Level Separation

Use `"secondaryPreferred"` for the main read path and `"primary"` for write-after-read patterns:

```javascript
// Default client: reads prefer secondaries
const client = new MongoClient(uri, {
  readPreference: "secondaryPreferred"
});

// For write-then-read sequences, explicitly use primary
const result = await collection.findOneAndUpdate(
  { _id: userId },
  { $inc: { loginCount: 1 } },
  {
    readPreference: new ReadPreference("primary"),
    returnDocument: "after"
  }
);
```

## Tag Sets for Targeted Distribution

Target reads to secondaries with specific tags while falling back to primary if none match:

```javascript
const readPref = new ReadPreference("secondaryPreferred", [
  { region: "us-east" }
]);

// Reads go to us-east secondaries; falls back to primary if none are available
await collection.find({}).withReadPreference(readPref).toArray();
```

## Use Cases

`"secondaryPreferred"` is well-suited for:

- **User-facing reads** in read-heavy applications (product catalog, feed, search)
- **API servers** that can tolerate a few hundred milliseconds of staleness
- **Multi-region deployments** where reads should hit local replicas
- **Reducing primary CPU** without sacrificing availability

## What to Avoid

Avoid using `"secondaryPreferred"` for:

- Reads immediately after writes where consistency is required
- Cart/checkout flows where inventory must be accurate
- Any path that depends on seeing the very latest committed data

## Summary

Read preference `"secondaryPreferred"` offers the best balance of read scalability and availability: it routes reads to secondaries by default and automatically falls back to the primary when no secondaries are reachable. Pair it with `maxStalenessSeconds` to bound acceptable lag and use tag sets to control geographic routing in multi-region replica sets.
