# How to Compare Strings in MongoDB Queries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, String, Query, Comparison, Aggregation

Description: Learn how to compare strings in MongoDB queries using equality, range operators, $strcasecmp, $cmp, and case-insensitive collation for accurate string matching.

---

String comparison in MongoDB is case-sensitive by default and uses binary collation (byte-by-byte comparison of UTF-8 values). Understanding the available comparison tools - from basic equality to collation-aware sorting - lets you write queries that return the results users expect.

## Basic String Equality

Exact string matching uses the implicit equality operator.

```javascript
db.users.find({ status: "active" });
db.users.find({ status: { $eq: "active" } });
```

Both forms are equivalent. String comparisons are case-sensitive, so `"Active"` and `"active"` are different values.

## Range Comparisons on Strings

Use `$gt`, `$gte`, `$lt`, and `$lte` with strings. MongoDB compares strings lexicographically by their UTF-8 byte values.

```javascript
// Find usernames that come after "m" alphabetically
db.users.find({ username: { $gt: "m" } });

// Find products with codes in a specific alphabetical range
db.products.find({
  code: { $gte: "A100", $lte: "A999" }
});
```

Lexicographic order places uppercase letters before lowercase in ASCII, so `"Z" < "a"` in a binary comparison.

## Case-Insensitive Comparison with Collation

The cleanest way to do case-insensitive comparisons is to specify a collation with `strength: 2` on the query.

```javascript
db.users.find(
  { username: "alice" },
  {}
).collation({ locale: "en", strength: 2 });
```

A `strength` of 2 ignores case differences. This query matches `"alice"`, `"Alice"`, and `"ALICE"`.

To make index-supported case-insensitive queries efficient, create the index with the same collation.

```javascript
db.users.createIndex(
  { username: 1 },
  { collation: { locale: "en", strength: 2 } }
);
```

## $strcasecmp in Aggregation

The `$strcasecmp` operator compares two strings case-insensitively in an aggregation pipeline. It returns `-1`, `0`, or `1`.

```javascript
db.products.aggregate([
  {
    $project: {
      nameMatch: {
        $strcasecmp: ["$name", "widget pro"]
      }
    }
  },
  {
    $match: { nameMatch: 0 }
  }
]);
```

A result of `0` means the strings are equal when case is ignored.

## $cmp for General Comparison

`$cmp` compares two values of any type and returns `-1`, `0`, or `1`. It is case-sensitive for strings.

```javascript
db.products.aggregate([
  {
    $project: {
      isAfterM: {
        $cmp: ["$name", "M"]
      }
    }
  }
]);
```

## Comparing Strings with $expr in Filters

Use `$expr` with `$eq` or `$strcasecmp` to filter documents based on computed string comparisons.

```javascript
db.orders.find({
  $expr: {
    $eq: [{ $toLower: "$status" }, "pending"]
  }
});
```

Note that `$expr` with computed expressions will not use a standard index. Store a normalized lowercase version as a separate indexed field if this filter runs frequently.

## Summary

MongoDB string comparisons default to case-sensitive binary order. Use collation with `strength: 2` for case-insensitive equality and range queries, and pair it with a collation-aware index for performance. In aggregation pipelines, use `$strcasecmp` for case-insensitive comparison and `$cmp` for ordered comparison. Avoid heavy use of `$expr` with computed expressions on large collections without a supporting index.
