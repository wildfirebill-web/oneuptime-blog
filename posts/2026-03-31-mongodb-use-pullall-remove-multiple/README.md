# How to Use $pullAll to Remove Multiple Values from an Array in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array Operator, Update Operator, Document

Description: Learn how to use MongoDB's $pullAll operator to remove all occurrences of multiple specified values from an array in a single atomic operation.

---

## What Is the $pullAll Operator?

The `$pullAll` operator removes all instances of the specified values from an array. It accepts a list of values and performs exact equality matching on each element. This is a convenience operator when you have a known set of values to remove and don't need complex query conditions.

## Basic Syntax

```javascript
db.collection.updateOne(
  { <filter> },
  { $pullAll: { <arrayField>: [<value1>, <value2>, ...] } }
)
```

## Example: Removing Multiple Scores

```javascript
// Document: { _id: 1, scores: [85, 90, 78, 90, 95, 78] }

db.students.updateOne(
  { _id: 1 },
  { $pullAll: { scores: [90, 78] } }
)
// scores becomes: [85, 95]
```

Both 90 and 78 are removed, including all duplicate occurrences.

## Example: Removing Multiple Tags

```javascript
db.posts.updateOne(
  { _id: "post-5" },
  { $pullAll: { tags: ["outdated", "draft", "review"] } }
)
```

## Removing Strings from Arrays

```javascript
db.config.updateOne(
  { _id: "email-blocklist" },
  { $pullAll: { domains: ["spam.com", "fake.net", "junk.io"] } }
)
```

## Bulk Removal Across Multiple Documents

```javascript
db.products.updateMany(
  { type: "apparel" },
  { $pullAll: { unavailableSizes: ["XS", "XL"] } }
)
```

## Removing Objects from Arrays

`$pullAll` can remove object elements, but requires an exact match on the entire object including all fields and their order.

```javascript
db.inventory.updateOne(
  { _id: "warehouse-1" },
  {
    $pullAll: {
      items: [
        { sku: "ABC", quantity: 5 },
        { sku: "DEF", quantity: 3 }
      ]
    }
  }
)
```

Note that `{ sku: "ABC", quantity: 5 }` and `{ quantity: 5, sku: "ABC" }` may not match due to field ordering in BSON comparison.

## $pullAll vs $pull

| Feature | $pullAll | $pull |
|---------|----------|-------|
| Input | List of exact values | Value or query condition |
| Object matching | Exact full-document equality | Field-level query match |
| Supports $gt, $lt, etc. | No | Yes |
| Best use case | Remove known list of scalars | Remove by condition |

```javascript
// $pull with condition
db.items.updateOne({ _id: 1 }, { $pull: { values: { $lt: 50 } } })

// $pullAll with exact list
db.items.updateOne({ _id: 1 }, { $pullAll: { values: [10, 20, 30] } })
```

## When to Use $pullAll

- You have a static, predetermined list of values to remove
- You want cleaner syntax compared to `$pull` with `$in`

The equivalent using `$pull` and `$in`:

```javascript
db.items.updateOne(
  { _id: 1 },
  { $pull: { values: { $in: [10, 20, 30] } } }
)
```

Both achieve the same result for scalar values. `$pullAll` is simply more concise.

## Summary

The `$pullAll` operator efficiently removes all occurrences of multiple exact values from an array in a single atomic operation. It is best suited for removing a known list of scalar values. For conditional removal or flexible object matching, use `$pull` with query operators instead.
