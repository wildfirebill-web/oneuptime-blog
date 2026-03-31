# How to Use db.collection.getIndexes() in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Index, Performance, Schema Management, Administration

Description: Learn how to use db.collection.getIndexes() to inspect all indexes on a MongoDB collection and interpret the output for performance tuning.

---

## Overview of getIndexes()

`db.collection.getIndexes()` returns an array of documents describing all indexes on a collection. Each document includes the key pattern, name, uniqueness constraints, and other options. This is the first step in understanding the index landscape of a collection and identifying unused, redundant, or missing indexes.

## Basic Usage

```javascript
use myDatabase;
db.orders.getIndexes();
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
    "key": { "customerId": 1, "status": 1 },
    "name": "customerId_1_status_1"
  },
  {
    "v": 2,
    "key": { "createdAt": -1 },
    "name": "createdAt_-1",
    "expireAfterSeconds": 2592000
  }
]
```

The `_id_` index is always present and cannot be dropped. Other indexes appear in creation order.

## Reading the Output Fields

| Field | Meaning |
|---|---|
| `v` | Index version (2 is current) |
| `key` | Fields and sort direction (1 ascending, -1 descending) |
| `name` | Index name used in hint() and dropIndex() |
| `unique` | If true, enforces uniqueness |
| `sparse` | If true, only indexes documents with the field |
| `expireAfterSeconds` | TTL index setting |
| `partialFilterExpression` | Partial index filter |
| `2dsphereIndexVersion` | For geospatial indexes |

## Checking Indexes Programmatically in Node.js

```javascript
const { MongoClient } = require("mongodb");

async function listIndexes(uri, dbName, collectionName) {
  const client = new MongoClient(uri);
  await client.connect();
  const db = client.db(dbName);
  const indexes = await db.collection(collectionName).indexes();
  console.log(JSON.stringify(indexes, null, 2));
  await client.close();
}

listIndexes("mongodb://localhost:27017", "myDatabase", "orders");
```

In Python with PyMongo:

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017")
db = client["myDatabase"]
indexes = list(db["orders"].list_indexes())
for idx in indexes:
    print(idx)
```

## Using listIndexes Command Directly

The shell helper wraps the `listIndexes` command:

```javascript
db.runCommand({ listIndexes: "orders" });
```

This returns a cursor with the same data, useful when scripting against the admin interface.

## Identifying Redundant Indexes

An index on `{ a: 1, b: 1 }` makes a standalone index on `{ a: 1 }` redundant for queries that only filter on `a`, because the compound index can serve those queries too. Use `getIndexes()` output to spot this pattern:

```javascript
// Redundant - the compound index covers standalone a queries
{ "key": { "a": 1 }, "name": "a_1" }
{ "key": { "a": 1, "b": 1 }, "name": "a_1_b_1" }
```

You can safely drop `a_1` if all your queries that use `a` also benefit from the compound index prefix.

## Detecting TTL Indexes

TTL indexes have `expireAfterSeconds` in the output:

```javascript
db.sessions.getIndexes().filter(idx => idx.expireAfterSeconds !== undefined);
```

This is handy for auditing data retention policies.

## Comparing Indexes Across Environments

Export indexes from production and compare with staging:

```bash
mongosh --quiet --eval "db.orders.getIndexes()" myDatabase > prod_indexes.json
mongosh --quiet --eval "db.orders.getIndexes()" myDatabase > staging_indexes.json
diff prod_indexes.json staging_indexes.json
```

## Summary

`db.collection.getIndexes()` is the primary tool for auditing the index configuration of any MongoDB collection. By examining the key patterns, names, and options in the returned documents, you can identify redundant indexes to drop, TTL policies in effect, and missing compound indexes that could improve query performance.
