# How to Use $documentNumber for Row Numbering in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Window Function, DocumentNumber, Analytics

Description: Learn how to use $documentNumber inside $setWindowFields to assign sequential row numbers to MongoDB documents within a partition for pagination and ordering use cases.

---

The `$documentNumber` operator assigns a sequential integer starting at 1 to each document in a window partition, based on the sort order. Unlike `$rank`, it never skips numbers for ties - every document gets a unique sequential number within its partition.

## Basic Usage

Assign row numbers to all documents sorted by date:

```javascript
db.orders.aggregate([
  {
    $setWindowFields: {
      sortBy: { createdAt: 1 },
      output: {
        rowNumber: {
          $documentNumber: {}
        }
      }
    }
  }
])
```

Sample output:

```json
[
  { "orderId": "O001", "createdAt": "2024-01-01", "rowNumber": 1 },
  { "orderId": "O002", "createdAt": "2024-01-02", "rowNumber": 2 },
  { "orderId": "O003", "createdAt": "2024-01-02", "rowNumber": 3 },
  { "orderId": "O004", "createdAt": "2024-01-03", "rowNumber": 4 }
]
```

Notice that the two orders from 2024-01-02 get distinct numbers (2 and 3) because `$documentNumber` never ties.

## Difference from $rank and $denseRank

| Operator | Behavior on ties |
|---|---|
| `$rank` | Tied docs share a rank; next rank skips |
| `$denseRank` | Tied docs share a rank; next rank is consecutive |
| `$documentNumber` | Every doc gets a unique sequential number |

```javascript
db.scores.aggregate([
  {
    $setWindowFields: {
      sortBy: { score: -1 },
      output: {
        rank: { $rank: {} },
        denseRank: { $denseRank: {} },
        rowNum: { $documentNumber: {} }
      }
    }
  }
])
```

Output comparison:

```json
[
  { "name": "Alice", "score": 90, "rank": 1, "denseRank": 1, "rowNum": 1 },
  { "name": "Bob",   "score": 85, "rank": 2, "denseRank": 2, "rowNum": 2 },
  { "name": "Carol", "score": 85, "rank": 2, "denseRank": 2, "rowNum": 3 },
  { "name": "Dave",  "score": 80, "rank": 4, "denseRank": 3, "rowNum": 4 }
]
```

## Row Numbering Within Partitions

Use `partitionBy` to restart numbering at 1 for each group:

```javascript
db.transactions.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$accountId",
      sortBy: { date: 1 },
      output: {
        transactionNumber: {
          $documentNumber: {}
        }
      }
    }
  }
])
```

Each account's transactions are numbered independently starting from 1, making it easy to identify the first, second, and Nth transaction per account.

## Offset-Based Pagination with $documentNumber

Implement stable offset pagination by adding row numbers first, then filtering:

```javascript
const pageSize = 20;
const pageOffset = 40; // page 3, 0-indexed

db.products.aggregate([
  {
    $setWindowFields: {
      sortBy: { price: 1, _id: 1 },
      output: {
        rowNum: { $documentNumber: {} }
      }
    }
  },
  {
    $match: {
      rowNum: { $gt: pageOffset, $lte: pageOffset + pageSize }
    }
  }
])
```

## Using $documentNumber to Find the Nth Record Per Group

Retrieve the 2nd most recent order per customer:

```javascript
db.orders.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$customerId",
      sortBy: { createdAt: -1 },
      output: {
        orderNum: { $documentNumber: {} }
      }
    }
  },
  { $match: { orderNum: 2 } }
])
```

## Summary

`$documentNumber` provides stable, unique sequential numbering within `$setWindowFields` partitions. It is distinct from `$rank` and `$denseRank` in that it never produces ties - every document receives a different number. Use it for offset-based pagination, finding the Nth record per group, and anywhere a guaranteed unique position number is needed across a sorted document set.
