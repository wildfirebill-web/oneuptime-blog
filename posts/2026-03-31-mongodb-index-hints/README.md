# How to Use Index Hints in MongoDB with hint()

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Index, hint, Query Optimization, Performance

Description: Learn how to use MongoDB's hint() method to force the query planner to use a specific index, override suboptimal plan choices, and test index effectiveness.

---

## How hint() Works

MongoDB's query planner automatically selects the index it estimates will perform best for a given query. In most cases this works well, but sometimes the planner chooses a suboptimal index. The `hint()` method forces the query to use a specific index, bypassing the automatic planner selection.

Common reasons to use `hint()`:
- The planner selects a less efficient index for a complex query.
- You are testing whether a particular index improves performance.
- You want to benchmark multiple indexes against each other.
- You need a collection scan explicitly (hint `{ $natural: 1 }`).

```mermaid
flowchart TD
    A[Query Submitted]
    A --> B{hint() specified?}
    B -- Yes --> C[Use specified index directly]
    B -- No --> D[Query Planner evaluates candidates]
    D --> E[Planner selects winning plan]
    C --> F[Execute query]
    E --> F
```

## Syntax

You can specify the index by key pattern or by name:

```javascript
// By key pattern
db.collection.find(filter).hint({ field: 1 })

// By index name
db.collection.find(filter).hint("index_name")

// Force a collection scan
db.collection.find(filter).hint({ $natural: 1 })
```

## Examples

### Force a Specific Index

Given a collection with multiple indexes, force the query to use the `status_1` index:

```javascript
db.orders.createIndex({ status: 1 })
db.orders.createIndex({ customerId: 1, createdAt: -1 })

// Force use of the status index
db.orders.find({ status: "active", customerId: "cust_001" }).hint({ status: 1 })
```

### Force by Index Name

Using an index name is less fragile than using the key pattern if field names are long:

```javascript
db.orders.createIndex(
  { status: 1, createdAt: -1 },
  { name: "idx_status_createdAt" }
)

db.orders.find({ status: "active" }).hint("idx_status_createdAt")
```

### Force a Collection Scan

To test query performance without any index (or to process documents in natural insertion order):

```javascript
db.orders.find({ status: "active" }).hint({ $natural: 1 })
```

This is useful when comparing index-scan performance against a collection scan.

### Combine hint() with explain()

The primary use of `hint()` is testing. Combine it with `explain("executionStats")` to measure the performance of a specific index:

```javascript
// Test index A
const planA = await db.collection("orders").find({ status: "active" })
  .hint({ status: 1 })
  .explain("executionStats");

// Test index B
const planB = await db.collection("orders").find({ status: "active" })
  .hint({ customerId: 1, createdAt: -1 })
  .explain("executionStats");

console.log("Index A execution time:", planA.executionStats.executionTimeMillis, "ms");
console.log("Index B execution time:", planB.executionStats.executionTimeMillis, "ms");
```

### hint() in Aggregation Pipeline

Use `hint` as an option in the `aggregate()` command:

```javascript
db.orders.aggregate(
  [
    { $match: { status: "shipped" } },
    { $group: { _id: "$customerId", total: { $sum: "$amount" } } }
  ],
  { hint: { status: 1 } }
)
```

### hint() in Update and Delete

You can also hint indexes in `findOneAndUpdate`, `updateOne`, `updateMany`, and `deleteOne`:

```javascript
db.orders.updateOne(
  { status: "pending", customerId: "cust_001" },
  { $set: { status: "processing" } },
  { hint: { customerId: 1 } }
)
```

### Node.js Example

```javascript
const { MongoClient } = require("mongodb");

async function compareIndexes() {
  const client = new MongoClient("mongodb://localhost:27017");
  await client.connect();

  const orders = client.db("shop").collection("orders");

  // Create two indexes
  await orders.createIndex({ status: 1 }, { name: "idx_status" });
  await orders.createIndex({ customerId: 1, status: 1 }, { name: "idx_customer_status" });

  // Insert sample data
  for (let i = 0; i < 10000; i++) {
    await orders.insertOne({
      customerId: `cust_${Math.floor(Math.random() * 100)}`,
      status: Math.random() > 0.2 ? "completed" : "active",
      amount: Math.random() * 1000,
      createdAt: new Date()
    });
  }

  const filter = { status: "active", customerId: "cust_42" };

  // Test with hint on status index
  const plan1 = await orders.find(filter)
    .hint({ status: 1 })
    .explain("executionStats");

  // Test with hint on compound index
  const plan2 = await orders.find(filter)
    .hint({ customerId: 1, status: 1 })
    .explain("executionStats");

  // Compare
  console.log("idx_status:");
  console.log("  Time:", plan1.executionStats.executionTimeMillis, "ms");
  console.log("  Docs examined:", plan1.executionStats.totalDocsExamined);

  console.log("idx_customer_status:");
  console.log("  Time:", plan2.executionStats.executionTimeMillis, "ms");
  console.log("  Docs examined:", plan2.executionStats.totalDocsExamined);

  await client.close();
}

compareIndexes().catch(console.error);
```

### Plan Cache Clearing

If you suspect the query planner has cached a bad plan, clear the plan cache and let it re-evaluate:

```javascript
// Clear plan cache for a collection
db.orders.getPlanCache().clear()

// Clear plan cache for a specific query shape
db.orders.getPlanCache().clearPlansByQuery(
  { status: "active" },  // filter
  {},                    // projection
  { status: 1 }          // sort
)
```

## When Not to Use hint()

- **Do not use `hint()` in production queries permanently** unless you have profiled and confirmed it is better. The query planner adapts as data changes; a fixed hint may become suboptimal.
- **Do not use `hint()` to mask a missing index.** If the planner keeps choosing the wrong index, investigate why rather than hardcoding a hint.
- **Avoid `hint()` in application code** for routine queries. Reserve it for benchmarking and edge cases.

## Best Practices

- **Use `hint()` + `explain("executionStats")`** to benchmark multiple indexes side-by-side.
- **Use index name hints** instead of key patterns to avoid breaking when index definitions are refactored.
- **Prefer fixing the query or schema** over permanently using `hint()` in application code.
- **Clear the plan cache** if the planner appears stuck on a stale bad plan.
- **Test in a staging environment** before applying hints to production queries.

## Summary

`hint()` forces MongoDB to use a specific index for a query, bypassing the automatic query planner selection. Specify the index by key pattern (e.g., `{ status: 1 }`) or by name. It is most useful for benchmarking competing indexes with `explain("executionStats")` and for overriding the planner when it makes a consistently suboptimal choice. Avoid using `hint()` as a long-term fix in production code - prefer redesigning the query or index.
