# How to Use Read Concern 'linearizable' in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Read Concern, Consistency, Replica Set, Linearizability

Description: Learn how MongoDB's read concern 'linearizable' provides the strongest consistency guarantee by waiting for all prior writes to be acknowledged before returning data.

---

## What Is Read Concern "linearizable"?

Read concern `"linearizable"` is MongoDB's strongest consistency guarantee. It ensures that a read reflects all writes that completed before the read started - across the entire replica set - and that the data returned is durable (cannot be rolled back).

Unlike `"majority"` which reads data as of the majority commit point, `"linearizable"` guarantees that you are reading from a node that is currently the primary and has the most up-to-date view of all committed writes.

## When to Use "linearizable"

Use `"linearizable"` when:

- Implementing critical coordination logic (leader election, distributed locks)
- Enforcing strict read-your-writes guarantees across concurrent clients
- Building financial transaction systems that must see every confirmed write immediately
- The application cannot tolerate any possibility of stale reads

## The Guarantee

```text
Client A writes document (w:majority) at T=1
Client B reads with linearizable at T=2
Result: Client B is guaranteed to see Client A's write
```

This holds even if Client B's read is routed to the primary just before an election.

## Running a Linearizable Read

```javascript
db.accounts.findOne(
  { _id: "ACC-001" },
  { readConcern: { level: "linearizable" } }
)
```

Or with `find()`:

```javascript
db.accounts.find(
  { _id: "ACC-001" }
).readConcern("linearizable").toArray()
```

## Requiring maxTimeMS

Because `"linearizable"` waits to confirm primary status before returning, it can block indefinitely if the primary is undergoing election. Always combine it with `maxTimeMS` to set a timeout:

```javascript
db.accounts.findOne(
  { _id: "ACC-001" },
  {
    readConcern: { level: "linearizable" },
    maxTimeMS: 5000
  }
)
```

This returns an error if the read cannot complete within 5 seconds.

## Node.js Driver Example

```javascript
const { MongoClient } = require("mongodb");

async function readBalance(accountId) {
  const client = new MongoClient("mongodb://mongo1:27017/?replicaSet=rs0");
  await client.connect();

  const db = client.db("banking");
  const result = await db.collection("accounts").findOne(
    { _id: accountId },
    {
      readConcern: { level: "linearizable" },
      maxTimeMS: 10000
    }
  );

  await client.close();
  return result;
}
```

## How MongoDB Implements Linearizability

When a `"linearizable"` read arrives at the primary:

1. MongoDB checks that the node is still the primary (via a round-trip to a majority of the set)
2. After confirming primary status, it returns the read result
3. If the primary loses its election during this check, the read fails with an error

This is more expensive than `"majority"` because it requires an additional round of communication.

## Limitations

- Only valid on `find` and `findOne` commands issued to the **primary**
- Does not work with read preference `"secondary"` or `"nearest"`
- Cannot be used inside multi-document transactions (use `"majority"` there)
- Not available on sharded clusters through mongos (use `"snapshot"` in transactions instead)
- Higher latency than `"local"` or `"majority"` due to the primary confirmation round-trip

## Comparison of Read Concerns

```text
Level           Rollback Safe   Staleness    Latency     Primary Only
local           No              Possible     Lowest      No
majority        Yes             Possible     Moderate    No
linearizable    Yes             None         Highest     Yes
snapshot        Yes             None (txn)   Moderate    No
```

## Practical Example: Distributed Counter Check

Before issuing a ticket number, verify the counter is authoritative:

```javascript
const counter = await db.collection("counters").findOne(
  { _id: "ticketSeq" },
  { readConcern: { level: "linearizable" }, maxTimeMS: 5000 }
);

// Safe to use counter.value - it reflects all prior increments
const nextTicket = counter.value + 1;
```

## Summary

Read concern `"linearizable"` is the strongest consistency level in MongoDB. It guarantees that reads reflect all committed writes visible to a primary-confirmed node, with no risk of stale data or rollbacks. Always pair it with `maxTimeMS` to avoid indefinite blocking. Reserve it for scenarios where correctness is paramount and higher latency is acceptable - for most use cases, `"majority"` provides sufficient consistency at lower cost.
