# How to Use Read Preference "secondaryPreferred" in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Read Preference, Secondary, Replication, Scalability, Driver

Description: Learn how to use read preference secondaryPreferred in MongoDB to direct reads to secondaries with automatic fallback to the primary when needed.

---

## Introduction

Read preference `secondaryPreferred` routes reads to secondary replica set members when they are available, falling back to the primary only when no secondaries are reachable. It is the most commonly used preference for scaling read workloads because it maximizes secondary utilization while maintaining availability.

## How "secondaryPreferred" Works

- Reads go to a randomly selected secondary when at least one is available
- Falls back to the primary only if all secondaries are down
- Accepts replication lag - reads may return slightly stale data
- Never fails due to secondary unavailability (unlike `secondary`)

## Setting Read Preference in mongosh

```javascript
db.getMongo().setReadPref("secondaryPreferred");
db.products.find({ category: "electronics" }).readPref("secondaryPreferred");
```

## Setting in Node.js

```javascript
const { MongoClient, ReadPreference } = require("mongodb");

const client = new MongoClient(
  "mongodb://host1:27017,host2:27017,host3:27017/?replicaSet=rs0",
  { readPreference: ReadPreference.SECONDARY_PREFERRED }
);

await client.connect();
```

## Connection String Configuration

```javascript
const client = new MongoClient(
  "mongodb://host1,host2,host3/?replicaSet=rs0&readPreference=secondaryPreferred"
);
```

## Combining with maxStalenessSeconds

Limit acceptable staleness when reading from secondaries:

```javascript
const { ReadPreference } = require("mongodb");

const pref = new ReadPreference("secondaryPreferred", [], {
  maxStalenessSeconds: 90
});

const products = await collection.find({}).withReadPreference(pref).toArray();
```

## Typical Use Cases

Use `secondaryPreferred` for:
- Read-heavy product catalogs and content queries
- User profile reads where slight staleness is acceptable
- Dashboard and reporting queries
- Any workload where distributing reads improves throughput

```javascript
// Product search - secondaries handle the load
const searchResults = await db.collection("products").find(
  { $text: { $search: "wireless headphones" } },
  { readPreference: "secondaryPreferred" }
).limit(20).toArray();
```

## Comparison with "secondary"

| Scenario | secondary | secondaryPreferred |
|----------|-----------|-------------------|
| Secondary available | Routes to secondary | Routes to secondary |
| No secondaries | Error | Falls back to primary |
| Primary only setup | Error | Routes to primary |

## Summary

Read preference `secondaryPreferred` is the practical choice for scaling read workloads across replica sets. It sends reads to secondaries to reduce primary load while gracefully falling back to the primary if secondaries are unavailable. Combined with `maxStalenessSeconds`, it provides controlled trade-offs between performance and data freshness.
