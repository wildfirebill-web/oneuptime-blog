# How to Query Documents Where an Array Contains a Specific Value in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query, Array, Operator, Index

Description: Learn how to query MongoDB documents where an array field contains a specific value using equality, $in, and $all operators.

---

Querying documents based on array field content is one of the most common operations in MongoDB. MongoDB's array query operators let you search for exact values, multiple values, or combinations of values within arrays.

## Basic Array Membership Query

MongoDB's equality operator works directly with arrays. If you specify a scalar value, MongoDB checks whether any element in the array equals that value:

```javascript
// Find all posts tagged with "mongodb"
db.posts.find({ tags: "mongodb" })
```

This returns any document where `tags` is either the string `"mongodb"` or an array containing `"mongodb"` as one of its elements.

## Sample Data

```javascript
db.posts.insertMany([
  { title: "Intro to MongoDB", tags: ["mongodb", "nosql", "database"] },
  { title: "MongoDB Aggregation", tags: ["mongodb", "aggregation", "pipeline"] },
  { title: "Redis Caching", tags: ["redis", "caching", "performance"] }
])
```

## Using $in to Check Multiple Values

When you want to find documents where the array contains any one of several values, use `$in`:

```javascript
// Find posts tagged with either "mongodb" or "redis"
db.posts.find({ tags: { $in: ["mongodb", "redis"] } })
```

## Using $all to Require Multiple Values

When you need documents where the array contains all specified values, use `$all`:

```javascript
// Find posts tagged with both "mongodb" AND "aggregation"
db.posts.find({ tags: { $all: ["mongodb", "aggregation"] } })
```

## Querying Arrays of Objects

For arrays of embedded documents, use dot notation to match on a specific field:

```javascript
// Find orders containing a specific product
db.orders.find({ "items.productId": "SKU-1234" })
```

Or with a condition on the value:

```javascript
// Find carts with quantity > 5 for any item
db.carts.find({ "items.qty": { $gt: 5 } })
```

## Exact Array Match

To match an array exactly (same elements, same order), use equality directly:

```javascript
db.docs.find({ categories: ["tech", "news"] })
```

This is rarely useful in practice - prefer `$all` for unordered membership checks.

## Checking Array Size with $size

```javascript
// Find documents where the array has exactly 3 elements
db.posts.find({ tags: { $size: 3 } })
```

## Index Support for Array Queries

MongoDB automatically creates a multikey index when you index an array field:

```javascript
db.posts.createIndex({ tags: 1 })
```

A multikey index stores a separate index entry for each element, enabling efficient equality and `$in` lookups:

```javascript
db.posts.find({ tags: "mongodb" }).explain("executionStats")
// Look for IXSCAN with the tags index
```

## Negation: Arrays That Do Not Contain a Value

```javascript
// Find posts NOT tagged with "deprecated"
db.posts.find({ tags: { $ne: "deprecated" } })

// More explicit with $nin
db.posts.find({ tags: { $nin: ["deprecated", "archived"] } })
```

## Summary

To find documents where an array contains a specific value, use a simple equality condition on the array field - MongoDB handles the matching automatically. Use `$in` for multiple candidate values, `$all` when multiple values must all be present, and create a multikey index on the array field for performance. These patterns apply to arrays of scalars and to arrays of embedded documents using dot notation.
