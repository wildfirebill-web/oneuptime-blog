# How to Use $size to Query Arrays by Exact Length in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $size, Array Query, Query Operator, Array Length

Description: Learn how to use MongoDB's $size operator to find documents where an array field has an exact number of elements, plus alternatives for range-based length queries.

---

## What Is $size

The `$size` operator matches documents where an array field has exactly the specified number of elements.

Syntax:

```javascript
{ field: { $size: number } }
```

## Basic Examples

```javascript
// Find posts with exactly 3 tags
db.posts.find({ tags: { $size: 3 } })

// Find orders with exactly 1 item
db.orders.find({ items: { $size: 1 } })

// Find documents with an empty array
db.notifications.find({ messages: { $size: 0 } })
```

## Practical Use Cases

```javascript
// Users with exactly 2 phone numbers
db.users.find({ phoneNumbers: { $size: 2 } })

// Products with no images uploaded
db.products.find({ images: { $size: 0 } })

// Assignments with exactly the required 3 reviewers
db.pullRequests.find({ reviewers: { $size: 3 } })
```

## Limitation: No Range Queries with $size

`$size` only supports exact matches. You cannot write `{ $size: { $gt: 2 } }`:

```javascript
// INVALID - $size does not accept comparison operators
db.posts.find({ tags: { $size: { $gt: 2 } } })  // Error
```

## Range-Based Array Length Queries

To query "array has more than N elements", use the `$expr` operator with `$size` (aggregation expression):

```javascript
// Find posts with more than 3 tags
db.posts.find({
  $expr: { $gt: [{ $size: "$tags" }, 3] }
})

// Find posts with between 2 and 5 tags
db.posts.find({
  $expr: {
    $and: [
      { $gte: [{ $size: "$tags" }, 2] },
      { $lte: [{ $size: "$tags" }, 5] }
    ]
  }
})
```

Alternatively, use the `$exists` trick for "at least N elements":

```javascript
// Posts with at least 3 tags (tags[2] exists)
db.posts.find({ "tags.2": { $exists: true } })

// Posts with at most 2 tags (tags[2] does not exist)
db.posts.find({ "tags.2": { $exists: false } })
```

## Storing Array Length as a Field

For high-frequency length-based queries, store the array length as a separate field and index it:

```javascript
// Store item count when inserting/updating
await db.collection("orders").updateOne(
  { _id: orderId },
  {
    $push: { items: newItem },
    $inc: { itemCount: 1 }
  }
);

await db.collection("orders").createIndex({ itemCount: 1 });

// Now this is fast with an index
db.orders.find({ itemCount: { $gt: 5 } })
```

## $size in Aggregation

Use `$size` as an aggregation expression to compute array lengths:

```javascript
db.orders.aggregate([
  {
    $project: {
      orderId: 1,
      itemCount: { $size: "$items" }
    }
  },
  {
    $match: { itemCount: { $gt: 3 } }
  }
])
```

## Index Support

`$size` queries do not use regular indexes efficiently. The workarounds (index on length field, `$expr`, or `$exists` on index position) all have better index support.

## Common Mistakes

- Passing a non-integer or negative number to `$size` - causes an error.
- Using `$size: 0` expecting to find documents where the field is absent - it only matches empty arrays, not missing fields.
- Trying to use range operators with `$size` directly.

## Summary

The `$size` operator finds documents where an array has an exact number of elements. For range-based length queries (greater than, less than), use `$expr` with the aggregation `$size` expression, or the `$exists` trick on a specific array index. For the best performance on frequent length queries, store array length as a dedicated indexed field.
