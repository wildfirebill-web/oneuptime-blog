# How to Check if an Array Contains a Specific Element in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array, Aggregation

Description: Learn how to check if a MongoDB array contains a specific element using $in, $elemMatch, and the aggregation $in expression operator.

---

Checking array membership is one of the most frequent operations in MongoDB - finding documents where a tag exists, verifying a user has a specific role, or filtering by an item in a cart. MongoDB provides different tools depending on whether you are writing a query filter or an aggregation expression.

## Query Filter: Direct Equality

The simplest check uses direct equality. MongoDB automatically checks all array elements:

```javascript
// Find all posts that have the "mongodb" tag
db.posts.find({ tags: "mongodb" });
```

This matches documents where `tags` is `"mongodb"` (scalar) or where `tags` is an array containing `"mongodb"`. MongoDB tests each element transparently.

## Query Filter: $in for Multiple Values

To check membership against a set of possible values:

```javascript
// Find users with any of these roles
db.users.find({
  roles: { $in: ["admin", "moderator"] }
});
```

## Aggregation: $in as an Expression Operator

Inside aggregation pipelines (`$project`, `$addFields`, `$match` with `$expr`), `$in` works as an expression operator that returns a boolean:

```javascript
db.posts.aggregate([
  {
    $project: {
      title: 1,
      hasMongoDB: { $in: ["mongodb", "$tags"] }
    }
  }
]);
```

Note the argument order: `["value", "$arrayField"]` - the value to find comes first, the array comes second. This is the opposite of the query operator syntax.

## Using $in in $match with $expr

To filter documents in an aggregation based on array membership:

```javascript
db.posts.aggregate([
  {
    $match: {
      $expr: { $in: ["mongodb", "$tags"] }
    }
  }
]);
```

## Checking Subdocument Arrays with $elemMatch

When array elements are embedded documents, use `$elemMatch` to check a specific field:

```javascript
// Find orders that contain product "P-001"
db.orders.find({
  lineItems: { $elemMatch: { productId: "P-001" } }
});
```

For a single condition, the dot notation shorthand also works:

```javascript
db.orders.find({ "lineItems.productId": "P-001" });
```

## $in as Aggregation Expression in $addFields

Use `$in` to add a boolean computed field to each document:

```javascript
db.products.aggregate([
  {
    $addFields: {
      isOnSale: { $in: ["sale", "$tags"] },
      isFeatured: { $in: ["featured", "$tags"] }
    }
  }
]);
```

This adds `isOnSale: true/false` and `isFeatured: true/false` based on tag membership.

## Checking Membership in a Computed Array

`$in` can check against a dynamically computed array, not just a document field:

```javascript
db.users.aggregate([
  {
    $project: {
      username: 1,
      hasElevatedRole: {
        $in: [
          "$currentRole",
          ["admin", "superuser", "root"]
        ]
      }
    }
  }
]);
```

Here, `$currentRole` is a scalar field and the literal array is the second argument.

## Index Support

For query-level array membership checks, a multikey index supports `find` with equality or `$in`:

```javascript
db.posts.createIndex({ tags: 1 });
```

The aggregation `$in` expression operator does not directly use indexes in `$project`. Place index-backed `$match` stages early in the pipeline to reduce documents before computing expressions.

## Summary

For query filters, use direct equality (`tags: "value"`) or `$in` for multi-value checks. In aggregation pipelines, use the `$in` expression operator (`{ $in: ["value", "$arrayField"] }`) to return boolean membership results. Use `$elemMatch` for subdocument arrays. Always index array fields used in membership filters for query performance.
