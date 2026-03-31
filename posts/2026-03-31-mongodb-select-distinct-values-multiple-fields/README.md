# How to Select Distinct Values on Multiple Fields in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query, Aggregation, Distinct, Group

Description: Learn how to get distinct combinations of multiple fields in MongoDB using aggregation $group, since distinct() only supports a single field.

---

MongoDB's `distinct()` command only works with a single field. To get unique combinations of multiple fields - the equivalent of `SELECT DISTINCT field1, field2` in SQL - you need the aggregation pipeline with `$group`.

## The Limitation of distinct()

The `distinct()` command accepts only one field:

```javascript
// Works for single field
db.orders.distinct("status")
// Returns: ["pending", "shipped", "delivered", "cancelled"]

// Cannot pass multiple fields
db.orders.distinct("status", "region") // Error - this doesn't work
```

## Using $group for Multi-Field Distinct

Group by a composite `_id` of multiple fields:

```javascript
// Get distinct combinations of status and region
db.orders.aggregate([
  {
    $group: {
      _id: {
        status: "$status",
        region: "$region"
      }
    }
  }
])
```

Sample output:

```json
[
  { "_id": { "status": "shipped", "region": "west" } },
  { "_id": { "status": "pending", "region": "east" } },
  { "_id": { "status": "delivered", "region": "west" } }
]
```

## Adding Counts

Include a count of how many documents exist for each combination:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: { status: "$status", region: "$region" },
      count: { $sum: 1 }
    }
  },
  { $sort: { count: -1 } }
])
```

## Flattening the _id Fields

To make the output cleaner, use `$replaceRoot` or `$project`:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: { status: "$status", region: "$region" }
    }
  },
  {
    $replaceRoot: {
      newRoot: "$_id"
    }
  },
  { $sort: { region: 1, status: 1 } }
])
```

Output:

```json
[
  { "region": "east", "status": "pending" },
  { "region": "west", "status": "delivered" },
  { "region": "west", "status": "shipped" }
]
```

## With a Pre-Filter

Apply a `$match` before `$group` to restrict which documents participate:

```javascript
db.orders.aggregate([
  {
    $match: {
      year: 2024,
      totalAmount: { $gt: 100 }
    }
  },
  {
    $group: {
      _id: {
        customerId: "$customerId",
        productCategory: "$productCategory"
      },
      orderCount: { $sum: 1 },
      totalSpend: { $sum: "$totalAmount" }
    }
  },
  { $sort: { totalSpend: -1 } }
])
```

## Distinct Values from Nested Fields

For fields inside embedded objects:

```javascript
db.events.aggregate([
  {
    $group: {
      _id: {
        country: "$location.country",
        city: "$location.city"
      }
    }
  }
])
```

## Index Considerations

A compound index on the grouped fields can make the aggregation more efficient:

```javascript
db.orders.createIndex({ status: 1, region: 1 })
```

For large collections, consider using `allowDiskUse: true` if the group stage exceeds memory limits:

```javascript
db.orders.aggregate(
  [
    { $group: { _id: { status: "$status", region: "$region" } } }
  ],
  { allowDiskUse: true }
)
```

## Summary

To find distinct combinations of multiple fields in MongoDB, use the aggregation pipeline with `$group` and a composite `_id` object. Apply `$match` before `$group` to filter input documents, and use `$replaceRoot` to produce flat output documents. Index the grouped fields to improve performance on large collections.
