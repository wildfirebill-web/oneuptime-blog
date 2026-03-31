# How to Limit and Skip Results in MongoDB for Pagination

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Pagination, Limit, Skip, Query

Description: Learn how to implement offset-based pagination in MongoDB using limit() and skip(), including performance trade-offs and alternatives.

---

## Limiting Results with limit()

`limit(n)` restricts the number of documents returned by a query. It is essential for pagination and for preventing large result sets from overwhelming clients.

```javascript
// Return only 10 users
const users = await db.collection("users")
  .find({ status: "active" })
  .limit(10)
  .toArray();
```

## Skipping Results with skip()

`skip(n)` tells MongoDB to skip the first `n` matching documents and return the rest (up to any limit). Combined with `limit()`, this implements offset-based pagination.

```javascript
const pageSize = 10;
const pageNumber = 3; // 1-based

const users = await db.collection("users")
  .find({ status: "active" })
  .sort({ createdAt: -1 })
  .skip((pageNumber - 1) * pageSize)
  .limit(pageSize)
  .toArray();
```

Always pair `skip()` with a `sort()` to ensure a stable, consistent ordering across pages.

## Building a Pagination Helper

```javascript
async function paginate(collection, filter, sort, page, pageSize) {
  const skip = (page - 1) * pageSize;
  const [data, total] = await Promise.all([
    collection.find(filter).sort(sort).skip(skip).limit(pageSize).toArray(),
    collection.countDocuments(filter)
  ]);
  return {
    data,
    total,
    page,
    pageSize,
    totalPages: Math.ceil(total / pageSize)
  };
}

const result = await paginate(
  db.collection("products"),
  { category: "electronics" },
  { price: -1 },
  2,
  20
);
```

## The skip() Performance Problem

`skip()` is inefficient for large offsets. MongoDB must scan and discard `n` documents before returning results. Skipping to page 1000 with `pageSize=20` means scanning 19,980 documents.

```javascript
// Very slow for large skip values
.skip(50000).limit(20) // scans 50,000 documents to find 20
```

## Cursor-Based Pagination as an Alternative

For better performance on large datasets, use cursor-based (keyset) pagination:

```javascript
// First page - no cursor
const firstPage = await db.collection("posts")
  .find({ status: "published" })
  .sort({ createdAt: -1, _id: -1 })
  .limit(20)
  .toArray();

// Next page - use the last document's values as cursor
const lastDoc = firstPage[firstPage.length - 1];

const nextPage = await db.collection("posts")
  .find({
    status: "published",
    $or: [
      { createdAt: { $lt: lastDoc.createdAt } },
      { createdAt: lastDoc.createdAt, _id: { $lt: lastDoc._id } }
    ]
  })
  .sort({ createdAt: -1, _id: -1 })
  .limit(20)
  .toArray();
```

This approach uses an index to jump directly to the next page without scanning.

## When to Use skip() vs Cursor Pagination

| Factor | skip() | Cursor Pagination |
|--------|--------|------------------|
| Jump to arbitrary page | Yes | No |
| Performance at large offsets | Poor | Excellent |
| Stable results during writes | No | Yes |
| Implementation complexity | Low | Medium |

## Common Mistakes

- Using `skip()` without `sort()` - results are in arbitrary order.
- Not counting total documents for UI pagination controls.
- Using large `skip()` values in production without a performance plan.

## Summary

`limit()` and `skip()` together implement straightforward offset-based pagination in MongoDB. While simple to implement, `skip()` degrades at large offsets because it scans discarded documents. For high-performance pagination on large collections, switch to cursor-based pagination using the last document's sort key as the next page marker.
