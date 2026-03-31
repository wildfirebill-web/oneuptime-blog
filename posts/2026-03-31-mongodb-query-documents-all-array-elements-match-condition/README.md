# How to Query Documents Where All Array Elements Match a Condition in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query, Array, Aggregation, Operator

Description: Learn how to find MongoDB documents where every element in an array field satisfies a condition using $not, $elemMatch, and aggregation $filter.

---

MongoDB does not have a built-in operator that directly asserts "all array elements match this condition." However, you can achieve this through logical negation or aggregation. Understanding the right approach depends on the complexity of your condition.

## The Logical Negation Approach

The most efficient method is double-negation: a document has all elements matching a condition if it has no element that fails the condition.

```javascript
// Find products where ALL prices are below 100
db.products.find({
  prices: { $not: { $elemMatch: { $gte: 100 } } }
})
```

This reads as: "find documents where no element in `prices` is >= 100." If no element fails, then all elements pass.

## Sample Data Setup

```javascript
db.shipments.insertMany([
  {
    orderId: "A001",
    packages: [
      { weight: 2, status: "delivered" },
      { weight: 3, status: "delivered" }
    ]
  },
  {
    orderId: "A002",
    packages: [
      { weight: 5, status: "delivered" },
      { weight: 1, status: "pending" }
    ]
  },
  {
    orderId: "A003",
    packages: []
  }
])
```

## Querying for All Elements Matching a String Condition

```javascript
// Find shipments where ALL packages have status "delivered"
db.shipments.find({
  packages: {
    $not: {
      $elemMatch: { status: { $ne: "delivered" } }
    }
  }
})
// Returns A001 only (A003 has empty array, also matches)
```

Note: documents with empty arrays also match because there are no elements that fail the condition.

## Filtering Out Empty Arrays

To exclude documents with empty arrays:

```javascript
db.shipments.find({
  "packages.0": { $exists: true },
  packages: {
    $not: { $elemMatch: { status: { $ne: "delivered" } } }
  }
})
```

## Using Aggregation $filter and $size

For more complex conditions, aggregation gives you precise control:

```javascript
db.shipments.aggregate([
  {
    $addFields: {
      nonDelivered: {
        $filter: {
          input: "$packages",
          as: "pkg",
          cond: { $ne: ["$$pkg.status", "delivered"] }
        }
      }
    }
  },
  {
    $match: {
      "packages.0": { $exists: true },
      "nonDelivered": { $size: 0 }
    }
  },
  {
    $project: { nonDelivered: 0 }
  }
])
```

## Using $allElementsTrue with $map

Another aggregation approach evaluates a condition across all elements:

```javascript
db.shipments.aggregate([
  {
    $match: {
      $expr: {
        $allElementsTrue: {
          $map: {
            input: "$packages",
            as: "pkg",
            in: { $eq: ["$$pkg.status", "delivered"] }
          }
        }
      }
    }
  }
])
```

Note: `$allElementsTrue` returns `true` for empty arrays, so add an array size check if needed.

## Performance Considerations

The negation approach (`$not: { $elemMatch: ... }`) can use a multikey index to find candidates for exclusion. However, these queries often require scanning more documents than simple array membership queries. Run `explain()` to verify plan efficiency:

```javascript
db.shipments.find({
  packages: { $not: { $elemMatch: { status: { $ne: "delivered" } } } }
}).explain("executionStats")
```

## Summary

To query documents where all array elements satisfy a condition, use the double-negation pattern: find documents where no element fails the condition via `$not` and `$elemMatch`. For complex multi-field conditions, use aggregation with `$filter` to count non-matching elements and `$match` where that count equals zero.
