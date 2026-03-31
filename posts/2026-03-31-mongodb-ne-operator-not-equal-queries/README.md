# How to Use $ne for Not Equal Queries in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query Operator, $ne, Filter, Exclusion

Description: Learn how to use MongoDB's $ne operator to match documents where a field does not equal a specified value, including null and missing field behavior.

---

## What Is $ne

The `$ne` (not equal) operator matches all documents where the field value is not equal to the specified value. It is the complement of `$eq`.

Syntax:

```javascript
{ field: { $ne: value } }
```

## Basic Example

Find all orders that are not in a "cancelled" status:

```javascript
db.orders.find({ status: { $ne: "cancelled" } })
```

This returns every document where `status` is not `"cancelled"`, including documents where `status` has any other value.

## $ne and Missing Fields

An important behavior: `$ne` also matches documents where the field does not exist at all. This differs from how `$eq` works.

```javascript
// Matches: status is not "cancelled" AND status field is missing
db.orders.find({ status: { $ne: "cancelled" } })
```

If you want to exclude documents with missing fields, combine `$ne` with `$exists`:

```javascript
db.orders.find({
  status: { $ne: "cancelled" },
  "status": { $exists: true }
})

// Cleaner with $and
db.orders.find({
  $and: [
    { status: { $exists: true } },
    { status: { $ne: "cancelled" } }
  ]
})
```

## Comparing Across Types

Like `$eq`, `$ne` is type-sensitive. The number `0` is not equal to the string `"0"`:

```javascript
// Matches docs where score is not 0 (number)
// Documents where score is "0" (string) are also returned
db.scores.find({ score: { $ne: 0 } })
```

## Using $ne in Nested Fields

```javascript
db.users.find({ "address.country": { $ne: "US" } })
```

## Combining $ne with Other Operators

```javascript
db.products.find({
  status: { $ne: "discontinued" },
  stock: { $gt: 0 },
  price: { $lt: 100 }
})
```

## $ne and Indexes

`$ne` queries are generally less efficient than equality queries because they cannot use an index to directly locate matching documents. MongoDB must scan all documents (or a full index range) and exclude the matching ones.

For better performance, consider rewriting exclusion queries as inclusion queries where possible:

```javascript
// Less efficient with $ne
db.orders.find({ status: { $ne: "cancelled" } })

// More efficient if you have a finite set of valid statuses
db.orders.find({ status: { $in: ["pending", "processing", "shipped", "delivered"] } })
```

## $ne in Aggregation Pipelines

In aggregation, use `$ne` as a comparison expression:

```javascript
db.orders.aggregate([
  {
    $project: {
      isNotCancelled: { $ne: ["$status", "cancelled"] }
    }
  }
])
```

## Common Mistakes

- Forgetting that `$ne` matches documents where the field is absent.
- Expecting `$ne` to be as fast as `$eq` - it typically requires more scanning.
- Using `$ne: null` expecting to filter out null values: this also matches documents where the field does not exist.

## Summary

The `$ne` operator matches documents where a field's value differs from the specified value, including documents where the field is missing entirely. Be cautious with index efficiency - `$ne` queries often scan more data than equality queries. When the set of valid values is known, use `$in` as a more performant alternative.
