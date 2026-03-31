# How to Find the Maximum and Minimum Values in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Max, Min, Analytics

Description: Learn how to find the maximum and minimum values of a field in MongoDB using the $max and $min aggregation operators, with examples for grouped and filtered queries.

---

## Using $max and $min in Aggregation

MongoDB provides `$max` and `$min` as accumulator operators within a `$group` stage. They return the highest and lowest values of a field across grouped documents.

## Finding Global Maximum and Minimum

To find the max and min across the entire collection, use `_id: null` to group everything together:

```javascript
db.products.aggregate([
  {
    $group: {
      _id: null,
      highestPrice: { $max: "$price" },
      lowestPrice: { $min: "$price" }
    }
  }
]);
```

Result:

```json
[{ "_id": null, "highestPrice": 4999.99, "lowestPrice": 0.99 }]
```

## Max and Min Per Category

Group by a field to get the extreme values per segment:

```javascript
db.products.aggregate([
  {
    $group: {
      _id: "$category",
      maxPrice: { $max: "$price" },
      minPrice: { $min: "$price" }
    }
  },
  { $sort: { maxPrice: -1 } }
]);
```

This returns the price range for each product category.

## Filtering Before Finding Extremes

Use `$match` to restrict which documents are considered:

```javascript
db.orders.aggregate([
  {
    $match: {
      status: "completed",
      region: "us-east"
    }
  },
  {
    $group: {
      _id: null,
      largestOrder: { $max: "$amount" },
      smallestOrder: { $min: "$amount" }
    }
  }
]);
```

## Using $max and $min with Dates

`$max` and `$min` work on date fields as well, returning the most recent and oldest dates:

```javascript
db.events.aggregate([
  {
    $group: {
      _id: "$userId",
      firstEvent: { $min: "$timestamp" },
      lastEvent: { $max: "$timestamp" }
    }
  }
]);
```

This is useful for finding the activity window per user.

## Finding the Document with the Maximum Value

`$max` returns the value, not the document. To retrieve the full document with the highest field value, use `$sort` and `$limit`:

```javascript
// Fastest approach using an index on price
db.products.find().sort({ price: -1 }).limit(1);
```

Or in aggregation when you need to combine this with a group:

```javascript
db.products.aggregate([
  { $sort: { price: -1 } },
  {
    $group: {
      _id: "$category",
      topProduct: { $first: "$$ROOT" }
    }
  }
]);
```

`$first` after a `$sort` gives you the document with the highest price per category.

## $max and $min on Nested Fields

Use dot notation for nested documents:

```javascript
db.sensors.aggregate([
  {
    $group: {
      _id: "$deviceId",
      peakTemp: { $max: "$readings.temperature" },
      minTemp: { $min: "$readings.temperature" }
    }
  }
]);
```

## Handling Null and Missing Values

`$max` and `$min` ignore null and missing field values. Only documents with non-null values for the target field contribute to the result.

## Summary

`$max` and `$min` in MongoDB aggregation efficiently find the extreme values of a numeric, date, or string field across a collection or per group. Combine them with `$match` to filter the scope and with `$sort` + `$first` when you need the full document corresponding to the extreme value.
