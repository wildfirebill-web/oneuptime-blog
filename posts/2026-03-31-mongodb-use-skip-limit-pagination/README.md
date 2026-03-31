# How to Use $skip and $limit for Pagination in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Pagination

Description: Learn how to implement pagination in MongoDB aggregation pipelines using $skip and $limit, including performance tradeoffs and cursor-based pagination as an alternative.

---

## What Are $skip and $limit?

The `$skip` stage skips the first N documents in the pipeline and passes the remaining documents to the next stage. The `$limit` stage restricts the pipeline output to the first N documents. Together, they implement offset-based pagination in aggregation queries.

## Basic Syntax

```javascript
db.collection.aggregate([
  { $sort: { <field>: 1 } },
  { $skip: <numberOfDocumentsToSkip> },
  { $limit: <pageSize> }
])
```

## Example: Basic Pagination

```javascript
const page = 3
const pageSize = 20

db.products.aggregate([
  { $sort: { name: 1 } },
  { $skip: (page - 1) * pageSize },
  { $limit: pageSize }
])
// Returns documents 41-60 (page 3)
```

## Getting Both Data and Total Count with $facet

```javascript
const page = 2
const pageSize = 10

db.articles.aggregate([
  { $match: { status: "published" } },
  {
    $facet: {
      data: [
        { $sort: { publishedAt: -1 } },
        { $skip: (page - 1) * pageSize },
        { $limit: pageSize }
      ],
      totalCount: [
        { $count: "count" }
      ]
    }
  }
])
```

This single aggregation returns both the page data and the total count for building pagination controls.

## Adding a $sort Before $skip

Always sort before paging to ensure consistent results across pages.

```javascript
db.users.aggregate([
  { $match: { active: true } },
  { $sort: { createdAt: -1 } },
  { $skip: 40 },
  { $limit: 20 }
])
```

Without a consistent sort, the order of documents between pages is undefined.

## Performance Characteristics of $skip

Large `$skip` values are expensive because MongoDB must scan and discard all skipped documents. Page 1000 with page size 20 requires scanning 19,980 documents before returning results.

```javascript
// Expensive: scans 10,000 documents to return page 501
db.events.aggregate([
  { $sort: { ts: -1 } },
  { $skip: 10000 },
  { $limit: 20 }
])
```

## Cursor-Based Pagination Alternative

For high-page-number performance, use a filter on the last-seen ID or timestamp instead of `$skip`.

```javascript
// Instead of $skip, filter by the last seen _id from the previous page
db.orders.aggregate([
  { $match: { _id: { $gt: lastSeenId } } },
  { $sort: { _id: 1 } },
  { $limit: pageSize }
])
```

This approach is O(log n) via the index regardless of page depth.

## $limit Without $skip

Use `$limit` alone to take only the top N results.

```javascript
db.products.aggregate([
  { $sort: { views: -1 } },
  { $limit: 10 }
])
// Top 10 most viewed products
```

## $skip in the $sort + $limit Optimization

MongoDB optimizes `$sort` + `$limit` into a top-N heap sort. Adding `$skip` breaks this optimization:

```javascript
// Optimized: $sort + $limit
[{ $sort: { score: -1 } }, { $limit: 10 }]

// Less optimized: $sort + $skip + $limit
[{ $sort: { score: -1 } }, { $skip: 20 }, { $limit: 10 }]
```

## Summary

The `$skip` and `$limit` stages implement offset-based pagination in MongoDB aggregation. Always pair them with a consistent `$sort`. For deep pagination where performance degrades, switch to cursor-based pagination using range filters on an indexed field. Use `$facet` to return both a page of data and the total count in a single pipeline.
