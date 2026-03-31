# How to Reduce RAM Usage with Projection in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Projection, Performance, Memory, Query Optimization

Description: Use MongoDB projection to return only the fields your application needs, reducing network transfer size and working set memory pressure on the server.

---

## Why Projection Reduces RAM Usage

Every document MongoDB returns to a query must travel through the server's working set (the in-memory cache). The larger each document, the more RAM is consumed to hold query results before they are sent to the client.

When you add a projection to a query, MongoDB trims each document to only the requested fields before returning it. This reduces:

- The amount of data loaded into the query result buffer.
- Network payload between the server and your application.
- Memory pressure on the application side.

## Basic Projection Syntax

Pass a second argument to `find()` specifying which fields to include (`1`) or exclude (`0`):

```javascript
// Include only the fields you need
db.users.find(
  { status: "active" },
  { name: 1, email: 1, _id: 0 }
);
```

The `_id` field is included by default; set it to `0` to suppress it.

You can also use exclusion to strip large fields:

```javascript
// Exclude a large blob field
db.products.find(
  { category: "electronics" },
  { description: 0, rawData: 0 }
);
```

Note: you cannot mix inclusion and exclusion in the same projection except for `_id`.

## Projecting Nested Fields

Use dot notation to project fields inside subdocuments:

```javascript
db.orders.find(
  { "shipping.status": "delivered" },
  { "customer.name": 1, "customer.email": 1, total: 1 }
);
```

Only the specified nested fields are returned, keeping the document small even if the parent object is large.

## Covered Queries: The Maximum Optimization

If your projection includes only indexed fields, MongoDB can satisfy the query entirely from the index without reading any documents from disk. This is called a covered query:

```javascript
// Index on { email: 1, status: 1 }
db.users.find(
  { status: "active" },
  { email: 1, status: 1, _id: 0 }
);
```

Verify coverage by checking that `explain("executionStats")` shows `totalDocsExamined: 0`.

## Projecting Array Slices

For documents with large arrays, use the `$slice` operator to return only the first N elements:

```javascript
db.posts.find(
  { published: true },
  { title: 1, comments: { $slice: 5 } }
);
```

This avoids loading the entire comments array when you only need a preview.

## Projection in the Aggregation Pipeline

In aggregation, use `$project` to reshape and reduce documents mid-pipeline:

```javascript
db.orders.aggregate([
  { $match: { status: "pending" } },
  { $project: { customerId: 1, total: 1 } },
  { $group: { _id: "$customerId", totalSpend: { $sum: "$total" } } }
]);
```

Placing `$project` early reduces the data each subsequent stage must process, lowering peak memory usage.

## Monitoring the Impact

Use `explain("executionStats")` before and after adding a projection to see the change in `totalDocsExamined` and query execution time:

```javascript
db.users.find({ status: "active" }, { name: 1 }).explain("executionStats");
```

For memory-intensive aggregations, watch the `usedDisk` field - if the pipeline previously spilled to disk but no longer does after adding `$project`, RAM savings are confirmed.

## Summary

MongoDB projection is one of the simplest and most effective ways to reduce RAM usage at query time. By returning only the fields your application actually uses, you shrink result sets, ease memory pressure on the server, and improve overall throughput. Combine projection with indexes for covered queries to achieve the maximum possible performance improvement.
