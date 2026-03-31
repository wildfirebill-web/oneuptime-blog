# How to Use $nin to Exclude Values in MongoDB Queries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query Operator, $nin, Filter, Exclusion

Description: Learn how to use MongoDB's $nin operator to match documents where a field does not equal any value in a specified list, including behavior with missing fields.

---

## What Is $nin

The `$nin` (not in) operator matches documents where the field value is not in the specified array. It is the inverse of `$in` and the array equivalent of `$ne`.

Syntax:

```javascript
{ field: { $nin: [value1, value2, value3] } }
```

## Basic Example

Find orders that are not in a terminal status:

```javascript
db.orders.find({ status: { $nin: ["cancelled", "refunded", "archived"] } })
```

Returns every document where `status` is not any of those three values.

## $nin and Missing Fields

Like `$ne`, the `$nin` operator also matches documents where the field does not exist. This can lead to unexpected results:

```javascript
// Also matches documents where "priority" field is absent
db.tickets.find({ priority: { $nin: ["low", "medium"] } })
```

To limit results to documents that have the field but with a non-excluded value:

```javascript
db.tickets.find({
  priority: { $exists: true, $nin: ["low", "medium"] }
})
```

## Excluding Specific User Roles

```javascript
const regularUsers = await db.collection("users").find({
  role: { $nin: ["admin", "superadmin", "system"] }
}).toArray();
```

## Using $nin with Array Fields

When the document field is an array, `$nin` matches documents where none of the elements in the field array appear in the specified list:

```javascript
// Matches posts tagged with none of the listed tags
db.posts.find({ tags: { $nin: ["deprecated", "draft", "hidden"] } })
```

## Combining $nin with Other Operators

```javascript
db.products.find({
  category: { $nin: ["accessories", "deprecated"] },
  stock: { $gt: 0 },
  price: { $lte: 200 }
})
```

## Performance Considerations

`$nin` queries are generally not as efficient as `$in` queries. MongoDB must scan all documents and exclude matches. When possible:

1. Ensure the field has an index.
2. Consider rewriting as an `$in` query with the set of acceptable values.

```javascript
// Less efficient
db.orders.find({ status: { $nin: ["cancelled", "archived"] } })

// More efficient (if you know the complete set of statuses)
db.orders.find({ status: { $in: ["pending", "processing", "shipped", "delivered"] } })
```

## $nin with an Empty Array

```javascript
// Matches all documents (nothing is excluded)
db.collection.find({ status: { $nin: [] } })
```

An empty exclusion list matches every document.

## Common Mistakes

- Forgetting that `$nin` also matches documents where the field is absent.
- Using `$nin` on unindexed fields in large collections without planning for the full scan cost.
- Expecting `$nin: ["value"]` to behave exactly like `$ne: "value"` - they are equivalent for scalar fields but differ for array fields.

## Summary

`$nin` matches documents where a field's value is not present in the specified list, and also matches documents where the field does not exist. Combine it with `$exists: true` when you need to exclude documents without the field. For large collections, ensure the field is indexed and consider whether an `$in` query on the known valid values would perform better.
