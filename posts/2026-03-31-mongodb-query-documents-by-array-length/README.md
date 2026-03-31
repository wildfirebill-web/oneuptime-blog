# How to Query Documents by Array Length in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array, Query

Description: Learn how to query MongoDB documents by array field length using $size, $expr with $size, and aggregation to filter by array count.

---

Querying documents based on the number of elements in an array field is a common requirement - finding users with no orders, products with at least three images, or posts with exactly five tags. MongoDB provides several approaches depending on whether you need an exact count or a range.

## Exact Array Length with $size

The `$size` operator matches documents where an array has exactly the specified number of elements:

```javascript
// Find products with exactly 3 tags
db.products.find({ tags: { $size: 3 } });

// Find users with exactly 0 orders (empty array)
db.users.find({ orders: { $size: 0 } });
```

The limitation of `$size` is that it only accepts an exact integer - it does not support range comparisons like `$gt` or `$lt`.

## Range Comparisons Using $expr and $size

For range queries (more than N elements, fewer than N elements), use `$expr` combined with the aggregation `$size` operator:

```javascript
// Find products with MORE than 2 tags
db.products.find({
  $expr: { $gt: [{ $size: "$tags" }, 2] }
});

// Find posts with FEWER than 5 comments
db.posts.find({
  $expr: { $lt: [{ $size: "$comments" }, 5] }
});

// Find orders with between 2 and 10 line items (inclusive)
db.orders.find({
  $expr: {
    $and: [
      { $gte: [{ $size: "$lineItems" }, 2] },
      { $lte: [{ $size: "$lineItems" }, 10] }
    ]
  }
});
```

The aggregation `$size` inside `$expr` computes the array length dynamically for each document.

## Handling Missing or Null Fields

If an array field might be missing or null, add a check before using `$size` to avoid errors:

```javascript
db.users.find({
  $expr: {
    $and: [
      { $isArray: "$orders" },
      { $gt: [{ $size: "$orders" }, 0] }
    ]
  }
});
```

`$isArray` returns `false` for missing or null fields, short-circuiting the `$size` call.

## Using Aggregation to Count and Filter

In an aggregation pipeline, use `$addFields` to compute the length, then `$match` on it:

```javascript
db.products.aggregate([
  {
    $addFields: {
      tagCount: { $size: "$tags" }
    }
  },
  {
    $match: {
      tagCount: { $gte: 3 }
    }
  },
  {
    $project: {
      name: 1,
      tagCount: 1,
      tags: 1
    }
  }
]);
```

This pattern is useful when you also want to include the computed count in the output.

## Index Strategy

Neither `$size` nor `$expr` with array size can use a standard index on the array field for these length queries. To make array length queries efficient at scale, store a denormalized count field alongside the array:

```javascript
db.products.updateMany(
  {},
  [{ $set: { tagCount: { $size: "$tags" } } }]
);

db.products.createIndex({ tagCount: 1 });
```

Then query by the count field directly:

```javascript
db.products.find({ tagCount: { $gte: 3 } });
```

Maintain the count field with every array update using `$inc` or by recomputing with an update pipeline.

## Summary

Use `$size` for exact array length matches in simple queries. For range comparisons, use `$expr` with the aggregation `$size` operator, and guard against missing fields with `$isArray`. For performance at scale, denormalize a count field alongside the array and index it separately.
