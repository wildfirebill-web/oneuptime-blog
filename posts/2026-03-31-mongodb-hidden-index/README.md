# What Is a Hidden Index in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Hidden Index, Index Management, Query Optimization, Database Performance

Description: A hidden index in MongoDB is an index that exists but is invisible to the query planner, letting you test index removal safely before dropping it.

---

## What Is a Hidden Index

A hidden index in MongoDB is a regular index that has been marked as invisible to the query planner. When an index is hidden, MongoDB still maintains it (keeping it up to date as documents are inserted, updated, or deleted), but the query planner will not use it to satisfy queries.

This feature is particularly useful when you want to evaluate the performance impact of dropping an index without permanently removing it. Instead of immediately dropping an index and hoping for the best, you can hide it first, observe the effect on query performance, and then decide whether to drop it or unhide it.

Hidden indexes were introduced in MongoDB 4.4.

## Why Hidden Indexes Are Useful

Before hidden indexes existed, dropping an index was a one-way operation. If query performance degraded after dropping an index, you had to rebuild it from scratch, which could take significant time on large collections.

With hidden indexes, the workflow becomes:

1. Hide the index
2. Monitor query performance
3. If performance degrades, unhide it immediately
4. If performance is unchanged or improved, drop it

This gives you a safe, reversible way to evaluate index usage without risking production query performance.

## How to Create a Hidden Index

You can create a new index as hidden from the start using the `hidden: true` option:

```javascript
db.orders.createIndex(
  { customerId: 1 },
  { hidden: true }
)
```

Or create a regular index and then hide it later:

```javascript
db.orders.createIndex({ customerId: 1 })
```

Then hide it with `hideIndex()`:

```javascript
db.orders.hideIndex({ customerId: 1 })
```

## How to Unhide an Index

To make a hidden index visible again to the query planner:

```javascript
db.orders.unhideIndex({ customerId: 1 })
```

You can also use `collMod` to toggle the hidden state:

```javascript
db.runCommand({
  collMod: "orders",
  index: {
    keyPattern: { customerId: 1 },
    hidden: true
  }
})
```

To unhide, set `hidden: false`:

```javascript
db.runCommand({
  collMod: "orders",
  index: {
    keyPattern: { customerId: 1 },
    hidden: false
  }
})
```

## Listing Indexes and Checking Hidden Status

To see all indexes on a collection, including whether they are hidden:

```javascript
db.orders.getIndexes()
```

Sample output:

```text
[
  {
    "v": 2,
    "key": { "_id": 1 },
    "name": "_id_"
  },
  {
    "v": 2,
    "key": { "customerId": 1 },
    "name": "customerId_1",
    "hidden": true
  }
]
```

The `hidden: true` field confirms the index is hidden.

## Behavior with the _id Index

The `_id` index cannot be hidden. It is always visible to the query planner and cannot be dropped or hidden.

## Using explain() to Confirm an Index Is Hidden

You can verify that a hidden index is not being used by checking the explain output of a query that would normally use it:

```javascript
db.orders.find({ customerId: "C123" }).explain("executionStats")
```

With the index hidden, the query planner will perform a COLLSCAN (collection scan) instead of an IXSCAN (index scan), confirming that the hidden index is not being considered.

## Practical Example: Evaluating an Unused Index

Suppose you have an index on `{ status: 1 }` and suspect it is never used. You can hide it and monitor your application:

```javascript
// Step 1: Hide the index
db.orders.hideIndex({ status: 1 })

// Step 2: Run your application workload and monitor performance
// Step 3: Check slow query logs for any regressions

// If no regressions after monitoring period, drop it
db.orders.dropIndex({ status: 1 })

// If regressions appear, unhide it immediately
db.orders.unhideIndex({ status: 1 })
```

## Hidden Indexes and Write Performance

Even when hidden, MongoDB still maintains the index on writes. This means:

- Insert, update, and delete operations still pay the write overhead for the hidden index
- Storage space is still used
- The only thing that changes is query planner visibility

This is an important consideration - hiding an index does not reduce write overhead or reclaim storage. It only affects read query planning.

## Summary

A hidden index in MongoDB is a powerful tool for safely evaluating whether an index is still needed. It remains maintained on writes but becomes invisible to the query planner, allowing you to simulate index removal without the risk of permanently losing it. The feature is available from MongoDB 4.4 onwards and is controlled via `hideIndex()`, `unhideIndex()`, and the `collMod` command.
