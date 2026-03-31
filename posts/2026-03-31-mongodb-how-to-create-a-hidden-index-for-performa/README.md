# How to Create a Hidden Index for Performance Testing in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, Hidden Index, Performance Testing, Database

Description: Learn how to create and use hidden indexes in MongoDB to safely test the impact of removing an index without deleting it, enabling risk-free index evaluation.

---

## Overview

Hidden indexes (introduced in MongoDB 4.4) allow you to hide an index from the query planner without dropping it. A hidden index is maintained on writes but is completely invisible to the query optimizer - enabling you to evaluate the performance impact of removing an index without the irreversible step of dropping it. If queries degrade, simply unhide the index to restore it.

## Creating a Hidden Index

Create a new index as hidden from the start:

```javascript
db.users.createIndex(
  { lastName: 1 },
  { hidden: true }
)
```

## Hiding an Existing Index

Hide an existing index using `collMod`:

```javascript
db.users.hideIndex("lastName_1")

// Or using collMod directly
db.runCommand({
  collMod: "users",
  index: {
    name: "lastName_1",
    hidden: true
  }
})
```

## Unhiding an Index

Restore the index to visible status:

```javascript
db.users.unhideIndex("lastName_1")

// Or using collMod
db.runCommand({
  collMod: "users",
  index: {
    name: "lastName_1",
    hidden: false
  }
})
```

## Workflow: Safe Index Removal Evaluation

Follow this process to safely evaluate whether an index can be dropped:

```text
Step 1: Identify a candidate index for removal
  - Use $indexStats to find indexes with low usage
  - Check index size and write overhead

Step 2: Hide the index
  - db.collection.hideIndex("index_name")

Step 3: Monitor performance
  - Watch query execution times
  - Monitor slow query logs
  - Check application error rates
  - Observe this for a representative period (hours to days)

Step 4: Decision
  - If performance is unaffected: drop the index permanently
  - If performance degrades: unhide the index immediately
```

## Checking Index Usage with $indexStats

Before hiding, identify unused or rarely used indexes:

```javascript
db.orders.aggregate([{ $indexStats: {} }])
```

Output includes `accesses.ops` (number of times the index was used) and `accesses.since` (when tracking started):

```javascript
// Example output
[
  { name: "email_1", accesses: { ops: 12453, since: ISODate("...") } },
  { name: "legacyField_1", accesses: { ops: 2, since: ISODate("...") } }
]
```

Low or zero access counts indicate candidates for hiding and potential removal.

## Viewing Hidden Indexes

Check whether an index is hidden with `getIndexes()`:

```javascript
db.users.getIndexes()
// Hidden indexes show "hidden": true in their definition
```

## Important Behaviors of Hidden Indexes

```text
- Hidden indexes ARE still maintained on write operations
  - Inserts, updates, and deletes still update the hidden index
  - Write performance impact is unchanged from a visible index

- Hidden indexes are NOT used by the query planner
  - Even with hint(), a hidden index is ignored
  - Exception: _id index cannot be hidden

- Hidden indexes can be unhidden instantly
  - No rebuild required - the index data is current
  - Visibility change takes effect immediately
```

## Forcing Hidden Index Usage with hint() for Testing

During the evaluation period, you can still force a hidden index to be used by specifying it in `hint()`:

```javascript
// Force the hidden index for a specific query to test it
db.users.find({ lastName: "Smith" }).hint("lastName_1").explain()
```

This allows you to test the index performance while it remains hidden from the regular query planner.

## Summary

Hidden indexes are an invaluable operational tool for safely evaluating index removal impact in MongoDB 4.4+. By hiding rather than dropping an index, you can observe real production query patterns without committing to the change. If performance degrades, unhiding restores the index instantly without a rebuild. Combine with `$indexStats` to identify low-usage index candidates and establish a disciplined, safe index deprecation workflow.
