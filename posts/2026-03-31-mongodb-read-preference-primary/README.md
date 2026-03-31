# How to Use Read Preference "primary" in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Read Preference, Replica Set, Consistency, Primary

Description: Learn how MongoDB's read preference "primary" works, why it is the default, and when it is appropriate to route all reads to the primary node.

---

## What Is Read Preference "primary"?

Read preference `"primary"` directs all read operations to the primary member of a replica set. It is the MongoDB default. No secondary is ever used for reads when this mode is active, regardless of latency or load.

## Why Primary Is the Default

The primary is the only replica set member guaranteed to have the most current data. Every write goes to the primary first. A read from the primary is the only way to guarantee you are seeing the latest committed state without additional configuration (such as `readConcern: "linearizable"`).

## Setting Read Preference to Primary

Because it is the default, you rarely need to set it explicitly. These are equivalent:

```javascript
db.orders.find({ status: "open" })

db.orders.find({ status: "open" }).readPref("primary")
```

In the connection string:

```text
mongodb://mongo1:27017,mongo2:27017,mongo3:27017/myapp?readPreference=primary&replicaSet=rs0
```

In the Node.js driver:

```javascript
const { MongoClient, ReadPreference } = require("mongodb");

const client = new MongoClient("mongodb://mongo1:27017/?replicaSet=rs0", {
  readPreference: ReadPreference.PRIMARY
});
```

## When Primary Read Preference Is Required

Certain operations must use the primary:

```javascript
// write operations always go to primary
db.orders.insertOne({ ... })
db.orders.updateOne({ ... })

// reads that must be consistent with just-written data
db.orders.findOne({ _id: newOrderId }).readPref("primary")

// operations that require current replica set state
db.adminCommand({ replSetGetStatus: 1 })
```

## What Happens if the Primary Is Unavailable

With `readPreference: "primary"`, if the primary is unreachable, **all reads fail**. MongoDB does not fall back to a secondary automatically:

```text
Primary crashes
  ->  Election begins (10-30 seconds)
  ->  All reads return error during election
  ->  New primary elected
  ->  Reads resume on new primary
```

Compare this to `"primaryPreferred"`, which falls back to secondaries during this window.

## Performance Implications

Because all reads go to the primary:

- The primary handles both reads and writes, increasing its load
- Secondaries are used only for replication, not for query distribution
- In read-heavy applications, the primary can become a bottleneck

This is fine for most write-heavy workloads where read volume is modest. For read-heavy workloads, consider `"secondary"` or `"secondaryPreferred"`.

## Consistency Characteristics

Reads from the primary with default `readConcern: "local"` return the latest data on the primary. However, there is still a small window where `w:1` writes are on the primary but not replicated, meaning they can be read and then rolled back after a failover.

For fully durable reads, pair with `readConcern: "majority"`:

```javascript
db.orders.find({ status: "open" })
  .readPref("primary")
  .readConcern("majority")
```

## Connection String Examples

```text
# Explicit primary preference
mongodb://mongo1,mongo2,mongo3/myapp?readPreference=primary&replicaSet=rs0

# Primary with fallback (different mode)
mongodb://mongo1,mongo2,mongo3/myapp?readPreference=primaryPreferred&replicaSet=rs0
```

## MongoDB Compass Configuration

In MongoDB Compass, set read preference under the connection options:
- Connection string: append `?readPreference=primary`
- GUI: Advanced Options -> Read Preference -> Primary

## Summary

Read preference `"primary"` routes all reads to the replica set's primary, providing the strongest consistency guarantee at the cost of no secondary read distribution. It is the default and appropriate for most write-centric applications where consistency matters more than read scalability. For read-heavy workloads or resilience during failover windows, consider `"primaryPreferred"` or `"secondary"` modes.
