# How to Use Read Preference "primary" in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Read Preference, Replication, Primary, Consistency, Driver

Description: Learn how to use read preference primary in MongoDB to route all read operations to the primary replica set member for the strongest consistency.

---

## Introduction

Read preference determines which replica set member MongoDB uses to serve a read operation. The `primary` read preference routes all reads to the current primary node. This is the default setting and provides the strongest consistency because the primary always has the most up-to-date data.

## How "primary" Read Preference Works

With `primary`:
- All reads go to the current primary
- If the primary is unavailable (e.g., during an election), reads fail
- No staleness risk - you always see the latest committed data
- Higher load on the primary compared to distributing reads to secondaries

## Setting Read Preference in mongosh

```javascript
db.getMongo().setReadPref("primary");
db.orders.find({ status: "active" });
```

Or per-query:

```javascript
db.orders.find({ status: "active" }).readPref("primary");
```

## Setting Read Preference in Node.js

```javascript
const { MongoClient, ReadPreference } = require("mongodb");

const client = new MongoClient("mongodb://host1:27017,host2:27017,host3:27017/?replicaSet=rs0", {
  readPreference: ReadPreference.PRIMARY
});

await client.connect();
const orders = client.db("mydb").collection("orders");

// Reads go to primary
const result = await orders.find({ status: "active" }).toArray();
```

## Setting in Connection String

```javascript
const client = new MongoClient(
  "mongodb://host1:27017,host2:27017,host3:27017/?replicaSet=rs0&readPreference=primary"
);
```

## When to Use "primary"

Use `primary` when:
- You need to read data immediately after writing it (read-your-write consistency)
- Your application cannot tolerate any replication lag
- You are performing writes and then reads in the same business transaction
- You are reading configuration or settings data that must be current

```javascript
// Write, then immediately read back - must use primary
await collection.insertOne({ orderId: "ORD-999", status: "new" });
const order = await collection.findOne(
  { orderId: "ORD-999" },
  { readPreference: "primary" }
);
```

## Failover Behavior

During a primary election, reads with `primary` preference will fail. Design your application to handle `MongoNotPrimaryError` with retry logic:

```javascript
async function readWithRetry(collection, filter, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try {
      return await collection.findOne(filter, { readPreference: "primary" });
    } catch (err) {
      if (i === retries - 1) throw err;
      await new Promise(r => setTimeout(r, 1000 * (i + 1)));
    }
  }
}
```

## Summary

Read preference `primary` ensures all reads go to the primary replica set member, providing the strongest consistency. It is the default and the correct choice for any operation requiring up-to-date data or read-after-write guarantees. The trade-off is that reads fail during a primary election and all read load stays on the primary node.
