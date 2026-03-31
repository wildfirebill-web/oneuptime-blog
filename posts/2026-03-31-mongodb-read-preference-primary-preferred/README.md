# How to Use Read Preference 'primaryPreferred' in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Read Preference, Replica Set, Availability, Failover

Description: Learn how MongoDB's read preference 'primaryPreferred' routes reads to the primary by default but falls back to secondaries during failover or primary unavailability.

---

## What Is Read Preference "primaryPreferred"?

Read preference `"primaryPreferred"` routes read operations to the primary when it is available. If the primary is unreachable (during an election or network partition), reads automatically fall back to a secondary. This makes it a more resilient alternative to `"primary"` while still preferring the most current data.

## When to Use primaryPreferred

Use `"primaryPreferred"` when:

- You want fresh data most of the time but prioritize availability during failovers
- Your application can tolerate slightly stale reads during the brief failover window
- You need to keep reading during a primary election (which lasts 10-30 seconds)

## Setting Read Preference to primaryPreferred

In `mongosh`:

```javascript
db.orders.find({ status: "open" }).readPref("primaryPreferred")
```

In the connection string:

```text
mongodb://mongo1:27017,mongo2:27017,mongo3:27017/myapp?readPreference=primaryPreferred&replicaSet=rs0
```

In the Node.js driver:

```javascript
const { MongoClient, ReadPreference } = require("mongodb");

const client = new MongoClient(
  "mongodb://mongo1:27017,mongo2:27017,mongo3:27017/?replicaSet=rs0",
  { readPreference: ReadPreference.PRIMARY_PREFERRED }
);
```

## Failover Behavior Comparison

```text
Scenario                    primary         primaryPreferred
Primary available           Reads primary   Reads primary
Primary election (10-30s)   Reads fail      Falls back to secondary
Primary down permanently    Reads fail      Reads secondary until new primary
```

## Staleness Risk During Failover

When `"primaryPreferred"` falls back to a secondary, reads may be slightly stale:

```text
1. Primary has latest write at oplog T=1000
2. Secondary has replicated up to T=995 (5 ops behind)
3. Primary goes down, failover begins
4. Reads from secondary: data as of T=995
5. New primary elected: reads current again
```

If your application cannot tolerate any staleness, use `"primary"` with retry logic instead.

## Adding maxStalenessSeconds

Limit which secondaries can serve reads during fallback by setting `maxStalenessSeconds`:

```javascript
db.orders.find({}).readPref("primaryPreferred", null, 30)
// Falls back only to secondaries with lag < 30 seconds
```

Or in the connection string:

```text
mongodb://mongo1,mongo2,mongo3/myapp?readPreference=primaryPreferred&maxStalenessSeconds=30&replicaSet=rs0
```

Minimum value is 90 seconds (enforced by the driver to avoid unrealistic expectations).

## Using Tag Sets With primaryPreferred

Combine with tag sets to restrict fallback reads to specific secondaries:

```javascript
const { ReadPreference } = require("mongodb");

const readPref = new ReadPreference("primaryPreferred", [
  { region: "us-east", role: "reporting" }
]);

const result = await collection.find({}).withReadPreference(readPref).toArray();
```

This falls back only to secondaries tagged with `region: "us-east"` and `role: "reporting"`.

## Practical Example: Web Application

A typical web application that wants high availability during deployments:

```javascript
const client = new MongoClient(
  "mongodb://mongo1:27017,mongo2:27017,mongo3:27017/?replicaSet=rs0",
  {
    readPreference: "primaryPreferred"
  }
);

// During a rolling restart of the primary:
// - Most reads continue to go to the primary
// - During the brief window when primary steps down, reads fall back to a secondary
// - Application continues serving requests without error
```

## Load Distribution Impact

Unlike `"secondary"` or `"secondaryPreferred"`, `"primaryPreferred"` does NOT spread read load across secondaries when the primary is healthy. The primary still handles all reads. Only during failover does this mode reduce primary load.

## Summary

Read preference `"primaryPreferred"` combines the consistency of primary reads with the availability of secondary fallback. It is ideal for applications that want fresh data but cannot accept read failures during the brief window of a primary election. Use `maxStalenessSeconds` to limit which secondaries are eligible during fallback, and consider tag sets to target specific secondary roles. For steady-state read distribution across secondaries, use `"secondaryPreferred"` instead.
