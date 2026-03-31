# How to Use $sortByCount to Rank and Count Groups in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Group By

Description: Learn how $sortByCount in MongoDB aggregation groups documents by an expression and returns them sorted by count descending, combining $group and $sort in one stage.

---

## What Is the $sortByCount Stage?

The `$sortByCount` stage groups documents by a specified expression and returns a count for each unique value, sorted from highest to lowest count. It is shorthand for a `$group` followed by a `$sort` on the count field, making it ideal for frequency analysis and ranking.

## Basic Syntax

```javascript
db.collection.aggregate([
  { $sortByCount: "<expression>" }
])
```

The output documents have two fields: `_id` (the group key) and `count`.

## Equivalent Longhand

```javascript
// $sortByCount is shorthand for:
db.collection.aggregate([
  { $group: { _id: "<expression>", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])
```

## Example: Most Common Product Categories

```javascript
db.products.aggregate([
  { $sortByCount: "$category" }
])
// Output:
// { _id: "electronics", count: 842 }
// { _id: "clothing", count: 654 }
// { _id: "books", count: 321 }
```

## Example: Most Frequent User Actions

```javascript
db.events.aggregate([
  { $match: { ts: { $gte: new Date(Date.now() - 86400000) } } },
  { $sortByCount: "$action" }
])
```

Counts all user actions in the last 24 hours, ranked by frequency.

## Using $sortByCount on a Field Value

```javascript
db.logs.aggregate([
  { $sortByCount: "$level" }
])
// { _id: "info", count: 12400 }
// { _id: "warn", count: 830 }
// { _id: "error", count: 142 }
```

## Using $sortByCount with $unwind

When fields are arrays, unwind first to count individual elements.

```javascript
db.articles.aggregate([
  { $unwind: "$tags" },
  { $sortByCount: "$tags" }
])
// Shows the most popular tags across all articles
```

## Using $sortByCount with an Expression

You can pass any valid aggregation expression, not just a field reference.

```javascript
db.orders.aggregate([
  {
    $sortByCount: {
      $dateToString: { format: "%Y-%m", date: "$createdAt" }
    }
  }
])
// Ranks months by number of orders
```

## Limiting Results

Chain `$limit` after `$sortByCount` to get only the top N groups.

```javascript
db.products.aggregate([
  { $sortByCount: "$category" },
  { $limit: 5 }
])
// Top 5 most common categories
```

## Combining with $match for Filtered Rankings

```javascript
db.orders.aggregate([
  { $match: { status: "completed", year: 2024 } },
  { $sortByCount: "$salesRepId" }
])
// Top performing sales reps by order count in 2024
```

## When to Use $sortByCount vs $group

| Situation | Use |
|-----------|-----|
| Need only count and sort by count | $sortByCount |
| Need additional accumulators (sum, avg, etc.) | $group |
| Need control over sort direction | $group + $sort |

## Summary

The `$sortByCount` stage is a concise aggregation shorthand that groups documents by an expression and returns the groups sorted by frequency in descending order. It is perfect for quick frequency analysis, tag clouds, action ranking, and any scenario where you need to know the most common values in your data.
