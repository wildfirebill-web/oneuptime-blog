# How to Calculate the Average of a Field in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Average, Analytics, Query

Description: Learn how to calculate the average of a numeric field in MongoDB using the $avg aggregation operator, with examples for simple averages and grouped averages.

---

## Using $avg in an Aggregation Pipeline

MongoDB provides the `$avg` accumulator for computing arithmetic means. It is used inside a `$group` stage:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: null,
      averageOrderValue: { $avg: "$amount" }
    }
  }
]);
```

Setting `_id: null` groups all documents together, producing a single document with the average across the entire collection.

Example result:

```json
[{ "_id": null, "averageOrderValue": 127.45 }]
```

## Average Per Group

To calculate the average per category or segment, provide a grouping field to `_id`:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$productCategory",
      avgAmount: { $avg: "$amount" }
    }
  },
  { $sort: { avgAmount: -1 } }
]);
```

This returns the average order value for each product category, sorted highest to lowest.

## Filtering Before Averaging

Use `$match` before `$group` to calculate the average over a subset of documents:

```javascript
db.orders.aggregate([
  {
    $match: {
      status: "completed",
      createdAt: { $gte: new Date("2025-01-01") }
    }
  },
  {
    $group: {
      _id: null,
      avgValue: { $avg: "$amount" }
    }
  }
]);
```

Always put `$match` before `$group` so MongoDB filters documents before aggregating, reducing the work.

## Average of a Nested Field

Use dot notation to average a field inside a subdocument:

```javascript
db.sensors.aggregate([
  {
    $group: {
      _id: "$deviceId",
      avgTemperature: { $avg: "$readings.temperature" }
    }
  }
]);
```

## Average with Rounding

`$avg` returns a floating-point number. Use `$round` to limit decimal places:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$region",
      avgAmount: { $avg: "$amount" }
    }
  },
  {
    $project: {
      avgAmount: { $round: ["$avgAmount", 2] }
    }
  }
]);
```

## Handling Null and Missing Values

`$avg` ignores `null` values and documents where the field does not exist. Only non-null numeric values contribute to the average:

```javascript
// Documents with amount: null or missing amount are skipped
db.orders.aggregate([
  {
    $group: {
      _id: null,
      avgAmount: { $avg: "$amount" }
    }
  }
]);
```

If you want to treat missing fields as zero, use `$ifNull`:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: null,
      avgAmount: {
        $avg: { $ifNull: ["$amount", 0] }
      }
    }
  }
]);
```

## Average Across Multiple Fields

Use `$avg` with an array expression to average several fields per document:

```javascript
db.scores.aggregate([
  {
    $project: {
      studentName: 1,
      avgScore: { $avg: ["$math", "$english", "$science"] }
    }
  }
]);
```

## Summary

The `$avg` operator in MongoDB aggregation calculates the arithmetic mean of a numeric field. Use `$group` with `_id: null` for a collection-wide average, or group by a field for per-segment averages. Always place `$match` before `$group` to filter documents early, and use `$round` to control decimal precision in the output.
