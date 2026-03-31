# How to Optimize Queries with Index Prefix Matching in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Index, Performance, Compound Index, Query Optimization

Description: Learn how MongoDB compound index prefix rules work so you can design indexes that serve multiple query shapes and avoid redundant single-field indexes.

---

A MongoDB compound index stores fields in a defined order. Any query that filters on a left-aligned prefix of that index can use it. Understanding this rule lets you cover multiple query shapes with a single index.

## What Is an Index Prefix?

For an index `{ a: 1, b: 1, c: 1 }`, the prefixes are:
- `{ a: 1 }` - prefix of length 1
- `{ a: 1, b: 1 }` - prefix of length 2
- `{ a: 1, b: 1, c: 1 }` - the full index

Any query filtering on `a`, or on `a` and `b`, or on `a`, `b`, and `c` can use this index.

## Creating a Compound Index

```javascript
db.orders.createIndex({ status: 1, region: 1, createdAt: 1 })
```

## Queries That Use This Index

```javascript
// Uses prefix { status: 1 }
db.orders.find({ status: "shipped" })

// Uses prefix { status: 1, region: 1 }
db.orders.find({ status: "shipped", region: "US" })

// Uses full index
db.orders.find({ status: "shipped", region: "US", createdAt: { $gte: ISODate("2026-01-01") } })
```

Verify with `explain`:

```javascript
db.orders.find({ status: "shipped" }).explain("queryPlanner")
// winningPlan.inputStage.stage should be "IXSCAN"
```

## Queries That Do NOT Use the Index

```javascript
// Missing "status" - not a prefix - COLLSCAN
db.orders.find({ region: "US" })

// Missing "status" - not a prefix - COLLSCAN
db.orders.find({ region: "US", createdAt: { $gte: ISODate("2026-01-01") } })
```

Skipping the leading field means MongoDB cannot use the index.

## Prefix Rule with Sort

Sort operations can also use index prefixes:

```javascript
// Sort on "status" alone uses the index
db.orders.find().sort({ status: 1 })

// Sort on "status, region" uses the index
db.orders.find().sort({ status: 1, region: 1 })
```

But a sort that skips the leading field does not use the index:

```javascript
// COLLSCAN + in-memory sort (no prefix)
db.orders.find().sort({ region: 1 })
```

## Eliminating Redundant Indexes

Because `{ status: 1, region: 1, createdAt: 1 }` already covers `{ status: 1 }` queries, you do not need a separate `{ status: 1 }` index. Remove redundant indexes to reduce write overhead:

```javascript
// Before: three separate indexes
db.orders.createIndex({ status: 1 })
db.orders.createIndex({ status: 1, region: 1 })
db.orders.createIndex({ status: 1, region: 1, createdAt: 1 })

// After: only the compound index is needed
db.orders.dropIndex("status_1")
db.orders.dropIndex("status_1_region_1")
```

## Choosing Field Order

Place the most selective fields first when the prefix rule allows it - this reduces the number of index entries scanned:

```javascript
// "userId" is more selective than "status"
db.orders.createIndex({ userId: 1, status: 1, createdAt: 1 })
```

## Summary

MongoDB compound indexes follow the prefix rule: a query must include the leading fields of the index from left to right to benefit from it. Design compound indexes so their prefixes match your most common query shapes, and drop single-field indexes that are made redundant by a compound index.
