# How to Store Money/Currency Values in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema, Decimal, Finance

Description: Learn the best practices for storing money and currency values in MongoDB, including Decimal128 vs integer cents, currency codes, and aggregation tips.

---

## Why Floating-Point Is Wrong for Money

Floating-point numbers (JavaScript `Number`, Python `float`) cannot represent all decimal fractions exactly. This leads to rounding errors that compound over many transactions:

```javascript
// Never do this
0.1 + 0.2  // 0.30000000000000004 - not 0.30
```

For financial data you need exact decimal arithmetic.

## Option 1: Store as Integer Cents (Recommended for Single Currency)

Multiply the amount by 100 and store as a plain 64-bit integer. This is fast, index-friendly, and avoids all floating-point issues:

```javascript
{
  _id: ObjectId("..."),
  description: "Widget purchase",
  amountCents: 4999,   // $49.99
  currency: "USD",
  createdAt: ISODate("2026-01-15T10:00:00Z")
}
```

Convert to a decimal string only for display:

```javascript
function formatMoney(cents, currency) {
  return new Intl.NumberFormat("en-US", {
    style: "currency",
    currency: currency
  }).format(cents / 100);
}
// formatMoney(4999, "USD") => "$49.99"
```

## Option 2: Use Decimal128 (Best for Multi-Currency / High Precision)

MongoDB's `Decimal128` BSON type stores 34 significant decimal digits with exact representation. It is the best native option when you need decimal precision directly in the database:

```javascript
// mongosh
db.invoices.insertOne({
  description: "Consulting fee",
  amount: NumberDecimal("1234.56"),
  currency: "EUR"
})
```

In Node.js with the official driver:

```javascript
const { Decimal128 } = require("mongodb");

await db.collection("invoices").insertOne({
  description: "Consulting fee",
  amount: Decimal128.fromString("1234.56"),
  currency: "EUR"
});
```

In Python:

```python
from bson import Decimal128

db.invoices.insert_one({
    "description": "Consulting fee",
    "amount": Decimal128("1234.56"),
    "currency": "EUR"
})
```

## Recommended Schema for Multi-Currency Applications

```json
{
  "_id": "inv-0001",
  "customerId": "cust-001",
  "lineItems": [
    {
      "description": "Widget A",
      "quantity": 3,
      "unitPriceCents": 2499,
      "currency": "USD"
    }
  ],
  "subtotalCents": 7497,
  "taxCents": 600,
  "totalCents": 8097,
  "currency": "USD",
  "exchangeRate": null,
  "createdAt": "2026-01-15T10:00:00Z"
}
```

Always store both the currency code (ISO 4217: `USD`, `EUR`, `GBP`) and the amount. Never store amounts from different currencies in the same numeric field without conversion.

## Aggregation with Integer Cents

Sum revenue per currency using `$group`:

```javascript
db.invoices.aggregate([
  {
    $group: {
      _id: "$currency",
      totalRevenueCents: { $sum: "$totalCents" }
    }
  }
])
```

## Aggregation with Decimal128

`$sum` and `$avg` work with `Decimal128` fields:

```javascript
db.invoices.aggregate([
  {
    $group: {
      _id: "$currency",
      totalRevenue: { $sum: "$amount" }
    }
  }
])
```

## Indexing for Financial Queries

Index frequently filtered fields like `currency`, `customerId`, and `createdAt`:

```javascript
db.invoices.createIndex({ customerId: 1, createdAt: -1 })
db.invoices.createIndex({ currency: 1, createdAt: -1 })
```

## Summary

Use integer cents for single-currency applications - it is simple, fast, and completely avoids floating-point errors. Use `Decimal128` when you need arbitrary decimal precision or are working with multi-currency data where exact decimal representation matters. Always store a currency code alongside every amount. Never mix currencies in a single numeric sum without explicit conversion.
