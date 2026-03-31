# How to Find the Largest Document in a MongoDB Collection

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Performance, Storage

Description: Learn how to find the largest document in a MongoDB collection using the aggregation pipeline and BSON size operators with practical examples.

---

## Why Document Size Matters

MongoDB enforces a 16 MB BSON document limit. Collections that store variable-length content - logs, embedded arrays, or binary data - can silently grow toward this limit. Identifying the largest documents lets you plan schema changes before hitting the cap.

## Using $bsonSize in the Aggregation Pipeline

MongoDB 4.4 introduced the `$bsonSize` expression, which returns the byte size of a document or value in BSON representation.

```javascript
db.posts.aggregate([
  {
    $project: {
      title: 1,
      docSize: { $bsonSize: "$$ROOT" }
    }
  },
  { $sort: { docSize: -1 } },
  { $limit: 5 }
])
```

This returns the five largest documents with their titles and sizes in bytes.

Sample output:

```json
[
  { "_id": "64abc1", "title": "Mega Post", "docSize": 524288 },
  { "_id": "64abc2", "title": "Long Article", "docSize": 102400 }
]
```

## Finding the Single Largest Document

```javascript
db.posts.aggregate([
  {
    $project: {
      docSize: { $bsonSize: "$$ROOT" }
    }
  },
  { $sort: { docSize: -1 } },
  { $limit: 1 }
])
```

To include the full document in the result, keep all fields in `$project`:

```javascript
db.posts.aggregate([
  {
    $addFields: {
      _docSize: { $bsonSize: "$$ROOT" }
    }
  },
  { $sort: { _docSize: -1 } },
  { $limit: 1 },
  { $unset: "_docSize" }
])
```

## Getting Summary Statistics

If you want to understand the size distribution across the whole collection:

```javascript
db.posts.aggregate([
  {
    $project: {
      docSize: { $bsonSize: "$$ROOT" }
    }
  },
  {
    $group: {
      _id: null,
      maxSize:  { $max: "$docSize" },
      minSize:  { $min: "$docSize" },
      avgSize:  { $avg: "$docSize" },
      totalDocs: { $sum: 1 }
    }
  }
])
```

## Finding Documents Above a Size Threshold

To identify all documents larger than 1 MB (1,048,576 bytes):

```javascript
db.posts.aggregate([
  {
    $project: {
      title: 1,
      docSize: { $bsonSize: "$$ROOT" }
    }
  },
  {
    $match: { docSize: { $gt: 1048576 } }
  },
  { $sort: { docSize: -1 } }
])
```

## Python Example

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017")
db = client["myapp"]

pipeline = [
    {"$project": {"docSize": {"$bsonSize": "$$ROOT"}}},
    {"$sort": {"docSize": -1}},
    {"$limit": 1},
]

result = list(db.posts.aggregate(pipeline))
if result:
    print(f"Largest document: {result[0]['_id']}, size: {result[0]['docSize']} bytes")
```

## Pre-4.4 Alternative

Before `$bsonSize` was available, you could use the JavaScript `Object.bsonsize()` function in the shell:

```javascript
// Shell only - not suitable for large collections
db.posts.find().forEach(doc => {
  print(doc._id, Object.bsonsize(doc));
});
```

This approach loads every document into the shell and is very slow on large collections. Prefer the aggregation pipeline approach whenever possible.

## Summary

Use `{ $bsonSize: "$$ROOT" }` inside an aggregation `$project` or `$addFields` stage to calculate the BSON size of each document. Sort descending and limit to one to find the largest document. Add a `$match` stage to filter documents above a specific byte threshold. Use the `$group` stage to compute max, min, and average sizes across the entire collection for capacity planning.
