# How to Use $all to Match Array Documents in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array Queries, Query Operators, Database, NoSQL

Description: Learn how to use the MongoDB $all operator to find documents where an array field contains all specified values, regardless of order or extra elements.

---

## What Is the $all Operator?

The `$all` operator in MongoDB matches documents where an array field contains all of the specified values. Unlike `$in` (which matches any), `$all` requires every listed value to be present in the field's array. The array may contain additional elements beyond those listed.

```javascript
{ field: { $all: [value1, value2, ...] } }
```

## Basic Example

Given a `products` collection with documents like:

```javascript
{ _id: 1, name: "Laptop", tags: ["electronics", "computing", "portable"] }
{ _id: 2, name: "Phone", tags: ["electronics", "mobile"] }
{ _id: 3, name: "Tablet", tags: ["electronics", "computing", "mobile"] }
```

To find products tagged with both `"electronics"` and `"computing"`:

```javascript
db.products.find({ tags: { $all: ["electronics", "computing"] } })
```

This returns documents 1 and 3, since document 2 doesn't have the `"computing"` tag.

## Using $all with a Single Element

When used with a single element, `$all` behaves like a direct equality match on an array:

```javascript
// These two are equivalent
db.products.find({ tags: { $all: ["mobile"] } })
db.products.find({ tags: "mobile" })
```

## Combining $all with Other Operators

You can combine `$all` with other query operators for more expressive queries:

```javascript
db.products.find({
  tags: { $all: ["electronics", "computing"] },
  price: { $lt: 1000 }
})
```

This finds affordable electronics that are also computing devices.

## Using $all with $elemMatch

For arrays of objects, use `$all` with `$elemMatch` to match documents where multiple complex conditions are all satisfied:

```javascript
db.courses.find({
  requirements: {
    $all: [
      { $elemMatch: { subject: "math", level: { $gte: 3 } } },
      { $elemMatch: { subject: "science", level: { $gte: 2 } } }
    ]
  }
})
```

This finds courses that require both a math level 3+ and a science level 2+ prerequisite.

## Aggregation Pipeline Usage

In aggregation, you can use `$setIsSubset` to achieve similar behavior:

```javascript
db.products.aggregate([
  {
    $match: {
      $expr: {
        $setIsSubset: [["electronics", "computing"], "$tags"]
      }
    }
  }
])
```

## Performance Considerations

- `$all` can use a multikey index on the queried array field.
- Placing the most selective values first in the `$all` array does not improve performance since all values must match.
- Create a multikey index on the array field to improve query performance:

```javascript
db.products.createIndex({ tags: 1 })
```

## Summary

The `$all` operator is a simple yet powerful tool for querying documents where array fields must contain every specified value. It supports nested `$elemMatch` conditions for complex object arrays and works well with multikey indexes. Use it when you need intersection-style array matching in MongoDB queries.
