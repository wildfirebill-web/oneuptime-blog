# How to Use Read Concern 'local' in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Read Concern, Consistency, Replica Set, Performance

Description: Learn how MongoDB's read concern 'local' works, when to use it for lowest-latency reads, and what rollback risk it carries in replica set deployments.

---

## What Is Read Concern "local"?

Read concern `"local"` is MongoDB's default read concern level. When a query uses `"local"`, MongoDB returns the most recent data available on the queried node at the time of the read - without waiting to confirm that the data has been replicated to or committed on other replica set members.

The name "local" reflects the fact that the data returned reflects the local state of the node you are reading from, which may not yet be acknowledged by a majority of the replica set.

## Default Behavior

Because `"local"` is the default, you do not need to set it explicitly. These two queries are equivalent:

```javascript
db.orders.find({ status: "open" })

db.orders.find({ status: "open" }).readConcern("local")
```

You can also set it explicitly in a session or at the driver level:

```javascript
const session = db.getMongo().startSession();
session.startTransaction({ readConcern: { level: "local" } });
```

## When "local" Is Appropriate

Read concern `"local"` is the right choice when:

- **Lowest latency is required** - no waiting for replication confirmation
- **Stale reads are acceptable** - for example, analytics dashboards or feed systems
- **Reading from the primary in a typical write flow** - the primary always has the latest data for writes that use `w:1` or higher
- **Inside multi-document transactions** - within a transaction, `"local"` gives consistent reads within that session

## The Rollback Risk

The key risk with `"local"` is that the returned data may be rolled back if the primary steps down before the data replicates to a majority:

```text
1. Client writes document A (w:1 on primary)
2. Client reads document A (readConcern: local) - returns data
3. Primary crashes before replicating A to majority
4. New primary is elected
5. Document A is rolled back - it never committed to the majority
6. Client holds data that no longer exists in the database
```

If your application cannot tolerate this scenario, use `"majority"` or `"linearizable"` instead.

## Performance Characteristics

`"local"` has no replication wait overhead. The read is served entirely from the node's in-memory or on-disk state:

```text
Read Concern     Latency    Rollback Risk
local            Lowest     Yes (on primary w:1 writes)
majority         Moderate   No
linearizable     Highest    No
```

## Using "local" With Secondary Reads

When combined with read preference `"secondary"`, `"local"` returns the secondary's current data which may lag behind the primary:

```javascript
db.stats.find({}).readPref("secondary").readConcern("local")
```

This is appropriate for reporting queries where some lag is acceptable and you want to offload reads from the primary.

## Node.js Driver Example

```javascript
const { MongoClient, ReadConcern } = require("mongodb");

async function run() {
  const client = new MongoClient("mongodb://mongo1:27017/?replicaSet=rs0");
  await client.connect();

  const db = client.db("myapp");
  const orders = await db
    .collection("orders")
    .find({ status: "open" })
    .withReadConcern(new ReadConcern("local"))
    .toArray();

  console.log(`Found ${orders.length} open orders`);
  await client.close();
}

run();
```

## Summary

Read concern `"local"` returns data immediately from the queried node without waiting for replication confirmation, providing the lowest possible read latency. It is the MongoDB default and is suitable when some risk of reading data that could be rolled back is acceptable. For scenarios requiring durability guarantees, pair writes with `w:majority` or switch to `readConcern: "majority"`.
