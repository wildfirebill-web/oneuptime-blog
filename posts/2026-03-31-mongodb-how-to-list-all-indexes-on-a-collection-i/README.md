# How to List All Indexes on a Collection in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, Index Management, Database

Description: Learn how to list all indexes on a MongoDB collection using getIndexes(), $indexStats, and db.stats() to audit, manage, and optimize your index strategy.

---

## Overview

Understanding what indexes exist on a collection is fundamental to query optimization and database maintenance. MongoDB provides several methods to inspect collection indexes: `getIndexes()` for index definitions, `$indexStats` for usage statistics, and collection `stats()` for size information. Together, these tools give a complete picture of your index landscape.

## Using getIndexes() to List Index Definitions

The `getIndexes()` method returns an array of all index definitions on a collection:

```javascript
db.orders.getIndexes()
```

Example output:

```javascript
[
  {
    "v": 2,
    "key": { "_id": 1 },
    "name": "_id_"
  },
  {
    "v": 2,
    "key": { "customerId": 1 },
    "name": "customerId_1"
  },
  {
    "v": 2,
    "key": { "status": 1, "createdAt": -1 },
    "name": "status_1_createdAt_-1"
  },
  {
    "v": 2,
    "key": { "email": 1 },
    "name": "email_1",
    "unique": true
  }
]
```

## Using db.collection.indexInformation()

A slightly different format showing index key specifications:

```javascript
db.orders.indexInformation()
```

Output is a plain object mapping index names to their key specifications.

## Using $indexStats to Get Usage Statistics

The `$indexStats` aggregation stage returns usage statistics for each index, including how many times each index has been accessed:

```javascript
db.orders.aggregate([{ $indexStats: {} }])
```

Example output:

```javascript
[
  {
    "name": "_id_",
    "key": { "_id": 1 },
    "host": "server:27017",
    "accesses": { "ops": 45231, "since": ISODate("2025-01-01T00:00:00Z") }
  },
  {
    "name": "legacyStatus_1",
    "key": { "legacyStatus": 1 },
    "host": "server:27017",
    "accesses": { "ops": 0, "since": ISODate("2025-01-01T00:00:00Z") }
  }
]
```

Note: `$indexStats` counts reset when `mongod` restarts. For accurate data, ensure sufficient uptime before using these counts for removal decisions.

## Finding Unused Indexes

Identify indexes with zero or very low access counts:

```javascript
db.orders.aggregate([
  { $indexStats: {} },
  { $match: { "accesses.ops": { $lt: 100 } } },
  { $project: { name: 1, key: 1, "accesses.ops": 1 } },
  { $sort: { "accesses.ops": 1 } }
])
```

## Checking Index Sizes

View the memory and disk footprint of each index:

```javascript
db.orders.stats().indexSizes
```

Example output:

```javascript
{
  "_id_": 2539520,
  "customerId_1": 4194304,
  "status_1_createdAt_-1": 8388608,
  "email_1": 2097152
}
```

Sizes are in bytes. Large indexes with low usage are prime candidates for removal.

## Listing All Indexes Across a Database

Iterate over all collections to audit indexes database-wide:

```javascript
db.getCollectionNames().forEach(function(collName) {
  const indexes = db.getCollection(collName).getIndexes()
  print(`\n--- ${collName} ---`)
  indexes.forEach(idx => print(`  ${idx.name}: ${JSON.stringify(idx.key)}`))
})
```

## Using Compass or Atlas for Visual Index Management

MongoDB Compass provides a visual interface to:
- View all indexes per collection
- See index type, size, and usage count
- Create and drop indexes without shell commands

In the Compass Indexes tab, you can filter and sort indexes by name, size, and type.

## Checking for Duplicate or Redundant Indexes

Look for indexes where one is a prefix of another:

```javascript
// If you have both these indexes, the first is redundant
// { status: 1 }
// { status: 1, createdAt: -1 }
// The compound index handles all queries the single-field index would handle
```

Script to find potential prefix redundancies:

```javascript
const indexes = db.orders.getIndexes().filter(i => i.name !== "_id_")
indexes.forEach(idx => {
  const keys = Object.keys(idx.key)
  indexes.forEach(other => {
    if (idx.name === other.name) return
    const otherKeys = Object.keys(other.key)
    const isPrefix = keys.every((k, i) => otherKeys[i] === k && idx.key[k] === other.key[k])
    if (isPrefix && keys.length < otherKeys.length) {
      print(`Potentially redundant: ${idx.name} is a prefix of ${other.name}`)
    }
  })
})
```

## Summary

Listing and analyzing indexes in MongoDB is essential for ongoing performance management. Use `getIndexes()` for index definitions, `$indexStats` for usage data, and `stats().indexSizes` for size information. Combine these to identify unused, redundant, or oversized indexes and maintain a lean, efficient index set. Regular index audits reduce write overhead, memory pressure, and storage usage across your MongoDB deployment.
