# How to Use Arithmetic Expressions in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Arithmetic, $add, $multiply

Description: Learn how to use $add, $subtract, $multiply, and $divide arithmetic expressions in MongoDB aggregation pipelines to compute derived fields from document data.

---

MongoDB aggregation pipelines support arithmetic expressions that compute values from document fields at query time. The four core operators - `$add`, `$subtract`, `$multiply`, and `$divide` - cover all basic math operations and can be nested for complex calculations.

## $add

`$add` sums two or more numbers. When used with dates, it adds milliseconds:

```js
db.orders.aggregate([
  {
    $project: {
      item: 1,
      totalWithTax: { $add: ["$subtotal", "$taxAmount"] },
      totalWithShipping: { $add: ["$subtotal", "$taxAmount", "$shippingCost"] }
    }
  }
]);
```

Adding days to a date (1 day = 86,400,000 ms):

```js
db.orders.aggregate([
  {
    $project: {
      dueDate: { $add: ["$orderDate", 7 * 24 * 60 * 60 * 1000] }
    }
  }
]);
```

## $subtract

`$subtract` takes exactly two arguments. When both are dates, it returns the difference in milliseconds:

```js
db.orders.aggregate([
  {
    $project: {
      profit: { $subtract: ["$revenue", "$cost"] },
      processingTimeMs: { $subtract: ["$shippedAt", "$createdAt"] }
    }
  }
]);
```

Convert milliseconds to hours:

```js
db.orders.aggregate([
  {
    $project: {
      processingHours: {
        $divide: [
          { $subtract: ["$shippedAt", "$createdAt"] },
          3600000
        ]
      }
    }
  }
]);
```

## $multiply

`$multiply` accepts two or more numbers:

```js
db.products.aggregate([
  {
    $project: {
      name: 1,
      totalRevenue: { $multiply: ["$price", "$unitsSold"] },
      discountedPrice: { $multiply: ["$price", 0.85] }
    }
  }
]);
```

## $divide

`$divide` takes exactly two arguments (dividend, divisor). Dividing by zero returns null:

```js
db.metrics.aggregate([
  {
    $project: {
      averageOrderValue: { $divide: ["$totalRevenue", "$orderCount"] },
      priceInDollars: { $divide: ["$priceInCents", 100] }
    }
  }
]);
```

## Combining Operators

Nest operators to build complex formulas. Here is a gross margin calculation:

```js
db.products.aggregate([
  {
    $project: {
      name: 1,
      grossMarginPct: {
        $multiply: [
          {
            $divide: [
              { $subtract: ["$revenue", "$cogs"] },
              "$revenue"
            ]
          },
          100
        ]
      }
    }
  }
]);
```

## Using Arithmetic in $group

Arithmetic expressions work inside `$group` accumulators too:

```js
db.sales.aggregate([
  {
    $group: {
      _id: "$region",
      totalProfit: {
        $sum: { $subtract: ["$revenue", "$cost"] }
      },
      avgMargin: {
        $avg: {
          $divide: [
            { $subtract: ["$revenue", "$cost"] },
            "$revenue"
          ]
        }
      }
    }
  }
]);
```

## Null and Missing Field Handling

If any argument is `null` or the field is missing, the entire expression returns `null`. Use `$ifNull` to provide defaults:

```js
db.orders.aggregate([
  {
    $project: {
      total: {
        $add: [
          { $ifNull: ["$subtotal", 0] },
          { $ifNull: ["$taxAmount", 0] }
        ]
      }
    }
  }
]);
```

## Summary

`$add`, `$subtract`, `$multiply`, and `$divide` let you compute derived numeric values directly in MongoDB aggregation without post-processing in application code. Nest them freely for complex formulas, use `$ifNull` to guard against missing fields, and combine them with `$group` to calculate aggregated metrics across document sets.
