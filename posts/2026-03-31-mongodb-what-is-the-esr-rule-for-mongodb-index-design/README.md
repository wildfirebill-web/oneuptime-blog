# What Is the ESR Rule for MongoDB Index Design

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Index, ESR Rule, Query Optimization, Performance

Description: Learn the ESR rule for ordering fields in compound MongoDB indexes to maximize query performance by placing Equality, Sort, and Range fields in the correct order.

---

## What Is the ESR Rule

ESR stands for Equality, Sort, Range - the recommended order for fields in a compound index. This ordering ensures MongoDB can use the index as efficiently as possible for queries that combine filtering, sorting, and range conditions.

## Why Field Order Matters in Compound Indexes

MongoDB reads index entries from left to right. Placing fields in the wrong order forces MongoDB to load more index data than necessary or fall back to an in-memory sort.

## The Three Field Types

- **Equality** - fields with exact match conditions: `{ status: "active" }`
- **Sort** - fields used in `.sort()` calls: `.sort({ createdAt: -1 })`
- **Range** - fields with comparison operators: `{ price: { $gte: 10, $lte: 100 } }`

## ESR Order Rule

Place fields in this order:
1. Equality fields first (most selective)
2. Sort fields second
3. Range fields last

## Example Without ESR (Suboptimal)

```javascript
// Query
db.products.find(
  { category: "electronics", price: { $gte: 50 } }
).sort({ createdAt: -1 })

// Index with wrong order (Range before Sort)
db.products.createIndex({ category: 1, price: 1, createdAt: -1 })
// This forces an in-memory sort on createdAt
```

## Example With ESR (Optimal)

```javascript
// Equality: category
// Sort: createdAt
// Range: price

// ESR-ordered index
db.products.createIndex({ category: 1, createdAt: -1, price: 1 })

// Query
db.products.find(
  { category: "electronics", price: { $gte: 50 } }
).sort({ createdAt: -1 })
```

Now MongoDB uses the index for both the equality filter and the sort, then applies the range filter efficiently.

## Verifying with explain()

```javascript
db.products.find(
  { category: "electronics", price: { $gte: 50 } }
).sort({ createdAt: -1 }).explain("executionStats")

// Good: "stage": "IXSCAN" with no "SORT" stage above it
// Bad: "stage": "SORT" above IXSCAN means in-memory sort
```

## Multi-Equality Fields

When multiple equality fields are present, put the most selective first:

```javascript
// category has 10 values, userId has millions of values
// userId is more selective

// ESR order with selectivity consideration
db.orders.createIndex({
  userId: 1,        // Equality (most selective)
  category: 1,      // Equality (less selective)
  createdAt: -1,    // Sort
  amount: 1         // Range
})
```

## Practical Query and Index Pairs

```javascript
// Query: find active users in US, sorted by signup date, with age range
db.users.find(
  {
    status: "active",      // Equality
    country: "US",         // Equality
    age: { $gte: 18 }      // Range
  }
).sort({ signupDate: -1 }) // Sort

// ESR Index
db.users.createIndex({
  status: 1,        // E
  country: 1,       // E
  signupDate: -1,   // S
  age: 1            // R
})
```

## Summary

The ESR rule says to put Equality fields first, Sort fields second, and Range fields last in compound indexes. This ordering lets MongoDB use the index efficiently for exact matches, eliminates in-memory sorts, and applies range filters after narrowing the result set. Validate with `explain()` to confirm no `SORT` stage appears above `IXSCAN`.
