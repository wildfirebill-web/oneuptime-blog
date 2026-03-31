# How to Use Aggregation Pipelines with PyMongo

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Python, PyMongo, Aggregation, Pipeline

Description: Learn how to build and run MongoDB aggregation pipelines with PyMongo to group, join, and transform documents for analytics and reporting.

---

## Overview

PyMongo's `aggregate` method executes a pipeline of stages that transform documents in a collection. Each stage takes the output of the previous stage as input, enabling powerful data transformations without moving data out of MongoDB.

## Basic Pipeline

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017/")
col = client["shop"]["orders"]

pipeline = [
    {"$match": {"status": "completed"}},
    {"$group": {
        "_id":   "$customerId",
        "total": {"$sum": "$amount"},
        "count": {"$sum": 1}
    }},
    {"$sort":  {"total": -1}},
    {"$limit": 10}
]

results = list(col.aggregate(pipeline))
for r in results:
    print(r["_id"], r["total"])
```

## Common Pipeline Stages

```text
$match   - Filter documents early to reduce pipeline input
$group   - Group by field and apply accumulators ($sum, $avg, $min, $max)
$project - Include, exclude, or compute fields
$sort    - Sort documents
$limit   - Limit result count
$skip    - Skip documents (for pagination)
$unwind  - Explode array field into separate documents
$lookup  - Left outer join with another collection
$addFields - Add new computed fields
$count   - Return document count as a field
$facet   - Run multiple sub-pipelines in parallel
```

## Joining with $lookup

```python
pipeline = [
    {"$match":  {"status": "pending"}},
    {
        "$lookup": {
            "from":         "customers",
            "localField":   "customerId",
            "foreignField": "_id",
            "as":           "customer"
        }
    },
    {"$unwind": "$customer"},
    {
        "$project": {
            "orderId":      1,
            "amount":       1,
            "customerName": "$customer.name",
            "customerEmail":"$customer.email"
        }
    }
]

orders = list(col.aggregate(pipeline))
```

## Grouping with Date Parts

```python
from datetime import datetime

pipeline = [
    {"$match": {"createdAt": {"$gte": datetime(2026, 1, 1)}}},
    {
        "$group": {
            "_id": {
                "year":  {"$year":  "$createdAt"},
                "month": {"$month": "$createdAt"}
            },
            "revenue": {"$sum": "$amount"},
            "orders":  {"$sum": 1}
        }
    },
    {"$sort": {"_id.year": 1, "_id.month": 1}}
]
```

## Allowing Disk Use

For large pipelines exceeding the 100 MB RAM limit:

```python
results = list(col.aggregate(pipeline, allowDiskUse=True))
```

## Iterating with a Cursor

For very large result sets, iterate the cursor instead of converting to list:

```python
cursor = col.aggregate(pipeline, allowDiskUse=True)
for doc in cursor:
    process(doc)
```

## Using $facet for Multi-Dimensional Analytics

```python
pipeline = [
    {"$match": {"status": "completed"}},
    {
        "$facet": {
            "byCategory": [
                {"$group": {"_id": "$category", "total": {"$sum": "$amount"}}}
            ],
            "byMonth": [
                {"$group": {"_id": {"$month": "$createdAt"}, "total": {"$sum": "$amount"}}}
            ]
        }
    }
]
```

## Summary

PyMongo's `aggregate` method accepts a list of pipeline stage dicts and returns a cursor. Use `$match` early to reduce data volume, `$group` for aggregation, `$lookup` for joins, and `$facet` for multi-dimensional analytics. Set `allowDiskUse=True` for pipelines that exceed the 100 MB memory limit. Iterate the cursor directly for memory-efficient processing of large result sets.
