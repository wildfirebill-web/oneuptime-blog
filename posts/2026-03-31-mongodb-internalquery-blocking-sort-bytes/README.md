# How to Configure internalQueryExecMaxBlockingSortBytes in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query, Performance, Configuration

Description: Learn how to configure internalQueryExecMaxBlockingSortBytes in MongoDB to control in-memory sort limits and prevent out-of-memory errors from large unsorted result sets.

---

When MongoDB executes a sort that cannot use an index, it performs an in-memory (blocking) sort. If the data being sorted exceeds a size limit, MongoDB returns an error rather than risk consuming excessive memory. The `internalQueryExecMaxBlockingSortBytes` parameter controls this limit.

## Default Value

The default value is **104,857,600 bytes (100 MB)**. If a blocking sort operation requires more than this amount of memory, MongoDB aborts the operation with:

```text
MongoServerError: Sort exceeded memory limit of 104857600 bytes
```

## Adjusting the Limit

**At runtime via mongosh:**

```javascript
db.adminCommand({
  setParameter: 1,
  internalQueryExecMaxBlockingSortBytes: 209715200  // 200 MB
});
```

**Via mongod.conf:**

```yaml
setParameter:
  internalQueryExecMaxBlockingSortBytes: 209715200
```

**Verify the current value:**

```javascript
db.adminCommand({
  getParameter: 1,
  internalQueryExecMaxBlockingSortBytes: 1
});
```

```text
{ internalQueryExecMaxBlockingSortBytes: 104857600, ok: 1 }
```

## When You Hit the Sort Limit

Before increasing the limit, check whether adding an index would eliminate the blocking sort:

```javascript
// Problematic query
db.orders.find({ status: "shipped" }).sort({ createdAt: -1 }).explain();
```

If the explain plan shows `SORT` with no index:

```text
{ "stage": "SORT", "sortPattern": { "createdAt": -1 } }
```

Create a compound index:

```javascript
db.orders.createIndex({ status: 1, createdAt: -1 });
```

Re-check - the plan should now show `IXSCAN` instead of `SORT`.

## Using allowDiskUse for Large Sorts

For aggregation pipelines, use `allowDiskUse` to spill sorts to disk instead of increasing the memory limit:

```javascript
db.orders.aggregate(
  [
    { $match: { status: "shipped" } },
    { $sort: { createdAt: -1 } }
  ],
  { allowDiskUse: true }
);
```

For `find()` queries in MongoDB 4.4+:

```javascript
db.orders
  .find({ status: "shipped" })
  .sort({ createdAt: -1 })
  .allowDiskUse(true);
```

## When to Increase internalQueryExecMaxBlockingSortBytes

Consider increasing the limit when:
- You have reporting or analytics queries that sort large datasets and adding indexes is not feasible
- You cannot use `allowDiskUse` due to performance constraints (disk spill is slow)
- You have sufficient RAM headroom on the server

**Example: Increasing to 256 MB for an analytics node:**

```javascript
db.adminCommand({
  setParameter: 1,
  internalQueryExecMaxBlockingSortBytes: 268435456
});
```

## Monitoring Sort Memory Usage

Track blocking sort metrics via `serverStatus`:

```javascript
db.adminCommand({ serverStatus: 1 }).metrics.query.sort
```

A high `spillToDisk` counter in aggregation pipeline sorts indicates either the memory limit is being hit or `allowDiskUse` is frequently triggered.

## Summary

`internalQueryExecMaxBlockingSortBytes` caps in-memory sorts at 100 MB by default, protecting MongoDB from OOM conditions. The preferred fix for sort errors is to add an index that supports the sort order. When that is not possible, use `allowDiskUse` for aggregations or increase the limit for nodes dedicated to analytics workloads.
