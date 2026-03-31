# How to Build a Cross-Tab (Pivot) Report in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pivot, Report, Analytics

Description: Learn how to build cross-tab pivot reports in MongoDB using the aggregation pipeline to transform row-based data into a columnar summary format.

---

## What Is a Cross-Tab Report

A cross-tab (pivot) report turns row-level data into a matrix where one dimension becomes column headers. For example, showing sales totals per product per month in a grid layout.

## Sample Data

Assume a `sales` collection with documents like:

```javascript
{ product: "Widget", month: "Jan", amount: 120 }
{ product: "Widget", month: "Feb", amount: 95 }
{ product: "Gadget", month: "Jan", amount: 200 }
{ product: "Gadget", month: "Feb", amount: 175 }
```

## Building the Pivot Query

Use `$group` with conditional sums to create columns dynamically:

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: "$product",
      Jan: {
        $sum: {
          $cond: [{ $eq: ["$month", "Jan"] }, "$amount", 0]
        }
      },
      Feb: {
        $sum: {
          $cond: [{ $eq: ["$month", "Feb"] }, "$amount", 0]
        }
      },
      Mar: {
        $sum: {
          $cond: [{ $eq: ["$month", "Mar"] }, "$amount", 0]
        }
      },
      total: { $sum: "$amount" }
    }
  },
  { $sort: { _id: 1 } }
])
```

## Dynamic Pivot Using $arrayToObject

For dynamic column names, group into key-value pairs and use `$arrayToObject`:

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: { product: "$product", month: "$month" },
      total: { $sum: "$amount" }
    }
  },
  {
    $group: {
      _id: "$_id.product",
      monthlySales: {
        $push: {
          k: "$_id.month",
          v: "$total"
        }
      }
    }
  },
  {
    $project: {
      product: "$_id",
      sales: { $arrayToObject: "$monthlySales" }
    }
  }
])
```

## Adding Row Totals

Combine both approaches to include row totals:

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: { product: "$product", month: "$month" },
      total: { $sum: "$amount" }
    }
  },
  {
    $group: {
      _id: "$_id.product",
      monthlySales: {
        $push: { k: "$_id.month", v: "$total" }
      },
      grandTotal: { $sum: "$total" }
    }
  },
  {
    $project: {
      product: "$_id",
      breakdown: { $arrayToObject: "$monthlySales" },
      grandTotal: 1
    }
  },
  { $sort: { product: 1 } }
])
```

## Handling Missing Months

Use `$ifNull` to fill in zeros for months with no data:

```javascript
{
  $project: {
    Jan: { $ifNull: ["$breakdown.Jan", 0] },
    Feb: { $ifNull: ["$breakdown.Feb", 0] },
    Mar: { $ifNull: ["$breakdown.Mar", 0] }
  }
}
```

## Summary

MongoDB's aggregation pipeline can produce pivot-style cross-tab reports using `$group` with conditional sums for static columns, or `$push` with `$arrayToObject` for dynamic column names. Adding a grand total field and null-filling with `$ifNull` produces a complete report suitable for dashboards or exports.
