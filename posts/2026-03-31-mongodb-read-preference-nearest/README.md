# How to Use Read Preference 'nearest' in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Read Preference, Replica Set, Latency, Multi-Region

Description: Learn how MongoDB's read preference 'nearest' routes reads to the lowest-latency replica set member regardless of whether it is primary or secondary.

---

## What Is Read Preference "nearest"?

Read preference `"nearest"` routes read operations to the replica set member with the lowest network round-trip time, regardless of whether that member is the primary or a secondary. It does not prefer one type over the other - it simply picks the fastest reachable node.

This mode is primarily designed for latency-sensitive reads in multi-region or geographically distributed deployments.

## How "nearest" Selects a Member

MongoDB measures the round-trip time (RTT) to each replica set member using heartbeat messages. When `"nearest"` is configured:

1. MongoDB identifies the member with the lowest measured RTT
2. All members within `localThresholdMS` of that RTT are considered candidates
3. One candidate is chosen randomly from the eligible set

Default `localThresholdMS` is 15ms, meaning all members within 15ms of the fastest are in the pool.

## Setting Read Preference to "nearest"

In `mongosh`:

```javascript
db.catalog.find({ category: "electronics" }).readPref("nearest")
```

In the connection string:

```text
mongodb://mongo1:27017,mongo2:27017,mongo3:27017/myapp?readPreference=nearest&replicaSet=rs0
```

In the Node.js driver:

```javascript
const { MongoClient, ReadPreference } = require("mongodb");

const client = new MongoClient(
  "mongodb://mongo1:27017,mongo2:27017,mongo3:27017/?replicaSet=rs0",
  { readPreference: ReadPreference.NEAREST }
);
```

## Multi-Region Deployment Example

Consider a 3-member replica set spread across three AWS regions:

```text
us-east-1  Primary  (writer)
eu-west-1  Secondary
ap-south-1 Secondary
```

Application instances in `ap-south-1` reading with `"nearest"` will automatically be served by the local secondary rather than making a cross-continent round trip to the primary:

```text
ap-south-1 client latency:
  primary (us-east-1):    180ms round trip
  secondary (eu-west-1):  220ms round trip
  secondary (ap-south-1):   2ms round trip  <- "nearest" selects this
```

## Adjusting localThresholdMS

Widen or narrow the candidate pool:

```javascript
const client = new MongoClient(uri, {
  readPreference: "nearest",
  localThresholdMS: 50  // include members within 50ms of fastest
});
```

A smaller value means stricter latency filtering. A larger value increases the randomization pool for load distribution.

## Staleness Risk With "nearest"

Because `"nearest"` may route to a secondary, reads can return data that is slightly behind the primary:

```javascript
// Limit staleness for nearest reads
const { ReadPreference } = require("mongodb");

const readPref = new ReadPreference("nearest", null, {
  maxStalenessSeconds: 120
});

await collection.find({}).withReadPreference(readPref).toArray();
```

## Tag Sets for Geographic Routing

Combine tags with `"nearest"` to pin reads to a specific region while still selecting the fastest node within that region:

```javascript
const readPref = new ReadPreference("nearest", [
  { region: "ap-south-1" }
]);

// Only considers members tagged region:ap-south-1, picks the fastest
await collection.find({}).withReadPreference(readPref).toArray();
```

## Nearest vs. Other Read Preferences

```text
Mode                 Primary preference  Secondary preference  Selection criteria
primary              Always primary      Never                 Role-based
primaryPreferred     Primary first       Fallback only         Role-based
secondary            Never primary       Always secondary      Role-based
secondaryPreferred   Secondary first     Fallback only         Role-based
nearest              Either              Either                Latency-based
```

## When "nearest" Is Most Valuable

Use `"nearest"` when:

- Application instances are co-located with replica set members in multiple regions
- Read latency is more important than data freshness
- Data being read changes slowly (product catalog, configuration, reference data)
- You are building a globally distributed application

Avoid `"nearest"` for:

- Checkout or payment flows where the latest inventory count is critical
- Reads immediately following writes where consistency matters

## Python Driver Example

```python
from pymongo import MongoClient, ReadPreference

client = MongoClient(
    "mongodb://mongo1:27017,mongo2:27017,mongo3:27017/?replicaSet=rs0",
    read_preference=ReadPreference.NEAREST
)

db = client["catalog"]
products = list(db.products.find({"category": "electronics"}))
```

## Summary

Read preference `"nearest"` provides the lowest possible read latency by routing to the closest replica set member, whether primary or secondary. It is the optimal choice for geographically distributed deployments where each region has a local replica. Pair it with tag sets to restrict the candidate pool and `maxStalenessSeconds` to bound acceptable data age.
