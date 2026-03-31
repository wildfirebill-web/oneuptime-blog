# How to Calculate the Sum of a Field Across Documents in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Sum, Analytics, Query

Description: Learn how to calculate the sum of a numeric field across all documents or per group in MongoDB using the $sum aggregation operator with practical examples.

---

## Using $sum in Aggregation

MongoDB's `$sum` accumulator adds numeric values together within a `$group` stage. It is the most common aggregation operation for totals, counts, and financial summaries.

## Total Sum Across All Documents

To sum a field across the entire collection, group with `_id: null`:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: null,
      totalRevenue: { $sum: "$amount" }
    }
  }
]);
```

Result:

```json
[{ "_id": null, "totalRevenue": 984250.75 }]
```

## Sum Per Group

Group by a field to get subtotals:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$region",
      totalSales: { $sum: "$amount" }
    }
  },
  { $sort: { totalSales: -1 } }
]);
```

This produces a total sales figure for each region, ordered highest first.

## Filtering Before Summing

Place `$match` before `$group` to sum only qualifying documents:

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
      _id: "$productId",
      totalQuantitySold: { $sum: "$quantity" },
      totalRevenue: { $sum: "$amount" }
    }
  }
]);
```

## Counting Documents with $sum

You can also use `$sum: 1` to count documents in each group (equivalent to `COUNT(*)`):

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$status",
      orderCount: { $sum: 1 }
    }
  }
]);
```

This is equivalent to a SQL `GROUP BY status` with `COUNT(*)`.

## Sum of a Computed Expression

You are not limited to summing raw field values. Use `$sum` with an expression:

```javascript
db.lineItems.aggregate([
  {
    $group: {
      _id: "$orderId",
      totalCost: {
        $sum: { $multiply: ["$quantity", "$unitPrice"] }
      }
    }
  }
]);
```

## Summing Nested Array Elements

To sum values across elements of an array in a single document, use `$reduce` or `$sum` in a `$project` stage:

```javascript
db.invoices.aggregate([
  {
    $project: {
      invoiceTotal: { $sum: "$lineItems.amount" }
    }
  }
]);
```

`$sum` applied to an array expression in `$project` sums the array elements per document.

## Null and Missing Field Handling

`$sum` treats `null` values and missing fields as zero:

```javascript
// Documents missing "amount" contribute 0 to the sum
db.orders.aggregate([
  { $group: { _id: null, total: { $sum: "$amount" } } }
]);
```

This behavior makes `$sum` safe to use even with incomplete data.

## Summary

The `$sum` operator in MongoDB aggregation totals numeric values across documents or within groups. Use `$group` with `_id: null` for a collection-wide total, group by a field for per-segment sums, and combine with `$match` to limit the scope. `$sum: 1` doubles as a document counter, making it a versatile tool for reporting pipelines.
