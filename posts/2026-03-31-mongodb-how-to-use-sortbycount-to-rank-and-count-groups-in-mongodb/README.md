# How to Use $sortByCount to Rank and Count Groups in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $sortByCount, Pipeline Stage, NoSQL

Description: Learn how to use MongoDB's $sortByCount aggregation stage to group documents by an expression, count occurrences, and return results sorted by count descending.

---

## What Is the $sortByCount Stage?

The `$sortByCount` stage is a shorthand aggregation stage that groups documents by an expression, counts the documents in each group, and returns the results sorted by count in descending order.

```javascript
{ $sortByCount: expression }
```

It is equivalent to:

```javascript
[
  { $group: { _id: expression, count: { $sum: 1 } } },
  { $sort: { count: -1 } }
]
```

## Basic Example

Count and rank documents by status:

```javascript
db.orders.aggregate([
  { $sortByCount: "$status" }
])
```

Output:

```javascript
[
  { _id: "completed", count: 850 },
  { _id: "pending", count: 312 },
  { _id: "cancelled", count: 87 },
  { _id: "refunded", count: 23 }
]
```

## Counting Tags or Categories

Useful for finding the most common values in a field:

```javascript
db.articles.aggregate([
  { $sortByCount: "$primaryCategory" }
])
```

## Counting After Unwinding Arrays

Combine with `$unwind` to count array element occurrences across all documents:

```javascript
db.products.aggregate([
  { $unwind: "$tags" },
  { $sortByCount: "$tags" }
])
```

This counts how many products have each tag, sorted from most to least popular.

## Using Expressions

`$sortByCount` accepts any valid aggregation expression, not just field references:

```javascript
// Count by derived value
db.events.aggregate([
  {
    $sortByCount: {
      $dateToString: { format: "%Y-%m", date: "$timestamp" }
    }
  }
])
// Groups by year-month and sorts by count
```

## Combining with $match for Filtered Rankings

```javascript
db.logs.aggregate([
  { $match: { level: "ERROR", timestamp: { $gte: new Date("2024-01-01") } } },
  { $sortByCount: "$errorCode" }
])
```

Find the most common error codes in recent logs.

## Practical Use Case - User Behavior Analysis

Find the most visited pages:

```javascript
db.pageViews.aggregate([
  { $match: { timestamp: { $gte: new Date("2024-06-01") } } },
  { $sortByCount: "$pagePath" }
])
```

## Practical Use Case - Top Search Keywords

```javascript
db.searchLogs.aggregate([
  { $sortByCount: { $toLower: "$query" } }
])
```

Converting to lowercase first ensures "MongoDB" and "mongodb" are counted together.

## Adding $limit for Top N

Get the top 10 most common values:

```javascript
db.products.aggregate([
  { $unwind: "$tags" },
  { $sortByCount: "$tags" },
  { $limit: 10 }
])
```

## Reshaping the Output

If you want to rename the `count` field or `_id`:

```javascript
db.orders.aggregate([
  { $sortByCount: "$country" },
  {
    $project: {
      _id: 0,
      country: "$_id",
      orderCount: "$count"
    }
  }
])
```

## Comparison with $group + $sort

For simple ranking, `$sortByCount` is more concise:

```javascript
// Verbose - $group then $sort
db.articles.aggregate([
  { $group: { _id: "$author", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])

// Concise - $sortByCount
db.articles.aggregate([
  { $sortByCount: "$author" }
])
```

Use `$group` directly when you need additional accumulators (like `$sum` of a value field, or `$push` to collect document IDs).

## Summary

`$sortByCount` is a concise aggregation stage that combines grouping and counting into a single readable step, automatically returning results sorted by frequency. It is ideal for frequency analysis, tag popularity, error ranking, and any scenario requiring "top N by occurrence" queries. When you also need additional aggregated metrics per group, use `$group` directly instead.
