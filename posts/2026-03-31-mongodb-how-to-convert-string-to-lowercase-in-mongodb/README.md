# How to Convert String to Lowercase in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, String, Aggregation, toLower, Expression, Pipeline

Description: Learn how to convert strings to lowercase in MongoDB using the $toLower aggregation operator in pipelines, projections, and computed fields.

---

## Introduction

MongoDB provides the `$toLower` aggregation expression operator to convert string field values to lowercase. This is useful for normalizing data, performing case-insensitive comparisons in aggregation pipelines, and generating slugs or usernames.

## Using $toLower in a Projection

Convert a field to lowercase in a query projection:

```javascript
db.users.aggregate([
  {
    $project: {
      username: 1,
      emailLower: { $toLower: "$email" }
    }
  }
]);
```

## Normalizing Data During a Migration

Convert existing email fields to lowercase permanently:

```javascript
db.users.find({}).forEach(doc => {
  if (doc.email !== doc.email.toLowerCase()) {
    db.users.updateOne(
      { _id: doc._id },
      { $set: { email: doc.email.toLowerCase() } }
    );
  }
});
```

For large collections, use `bulkWrite` for better performance:

```javascript
const ops = [];
db.users.find({ email: { $regex: /[A-Z]/ } }).forEach(doc => {
  ops.push({
    updateOne: {
      filter: { _id: doc._id },
      update: { $set: { email: doc.email.toLowerCase() } }
    }
  });
  if (ops.length === 1000) {
    db.users.bulkWrite(ops);
    ops.length = 0;
  }
});
if (ops.length > 0) db.users.bulkWrite(ops);
```

## Case-Insensitive Grouping

Group documents by a lowercased field value:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: { $toLower: "$status" },
      count: { $sum: 1 },
      totalAmount: { $sum: "$amount" }
    }
  },
  { $sort: { count: -1 } }
]);
```

This ensures "PENDING", "Pending", and "pending" are counted together.

## Generating Slugs

Create URL-friendly slugs from titles:

```javascript
db.articles.aggregate([
  {
    $addFields: {
      slug: {
        $replaceAll: {
          input: { $toLower: "$title" },
          find: " ",
          replacement: "-"
        }
      }
    }
  },
  {
    $merge: {
      into: "articles",
      whenMatched: "merge"
    }
  }
]);
```

## $toLower in a $match Stage

Filter using `$toLower` in a computed match (note: cannot use an index for this pattern - use collation indexes instead for performance):

```javascript
db.users.aggregate([
  {
    $match: {
      $expr: {
        $eq: [{ $toLower: "$email" }, "alice@example.com"]
      }
    }
  }
]);
```

For indexed lookups, prefer a collation index with `strength: 2` over expression-based matching.

## Summary

The `$toLower` aggregation operator converts strings to lowercase in MongoDB aggregation pipelines. It is useful for normalizing data, case-insensitive grouping, and slug generation. For high-performance case-insensitive queries, prefer collation indexes over `$toLower` in match stages, as expression-based matches cannot use standard indexes.
