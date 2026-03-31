# How to Use System Variables ($$ROOT, $$CURRENT, $$NOW) in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Variable

Description: Learn how to use MongoDB built-in system variables $$ROOT, $$CURRENT, $$NOW, $$CLUSTER_TIME, and $$REMOVE in aggregation pipelines.

---

MongoDB provides a set of system variables - prefixed with `$$` - that are available in all aggregation expressions without being defined. They give you access to the current document, the current time, and special sentinel values.

## $$ROOT - The Full Current Document

`$$ROOT` references the entire document as it exists at the current pipeline stage:

```javascript
// Wrap the document under a "data" key while adding metadata
db.events.aggregate([
  {
    $project: {
      data: "$$ROOT",
      processedAt: "$$NOW"
    }
  }
]);
```

`$$ROOT` is most commonly used with `$replaceRoot` and `$mergeObjects`:

```javascript
// After $lookup, flatten the joined data into the parent document
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  },
  {
    $replaceRoot: {
      newRoot: {
        $mergeObjects: [
          "$$ROOT",
          { $arrayElemAt: ["$customer", 0] }
        ]
      }
    }
  }
]);
```

## $$CURRENT - The Current Document in Context

`$$CURRENT` refers to the current document in most pipeline stages. In `$project` and `$addFields`, `$$CURRENT` and `$$ROOT` are equivalent. The distinction appears in stages like `$redact`, where `$$CURRENT` refers to the sub-document being evaluated:

```javascript
db.reports.aggregate([
  {
    $redact: {
      $cond: {
        if: { $in: ["classified", { $ifNull: ["$$CURRENT.labels", []] }] },
        then: "$$PRUNE",
        else: "$$DESCEND"
      }
    }
  }
]);
```

In `$redact`, `$$CURRENT` is the current node being visited as the operator recurses through the document tree.

## $$NOW - The Current Date and Time

`$$NOW` returns the current UTC date as a BSON Date. It is fixed to the moment the aggregation starts, so all documents in the pipeline see the same value:

```javascript
db.subscriptions.aggregate([
  {
    $addFields: {
      isExpired: { $lt: ["$expiresAt", "$$NOW"] },
      daysRemaining: {
        $divide: [
          { $subtract: ["$expiresAt", "$$NOW"] },
          86400000   // ms per day
        ]
      }
    }
  }
]);
```

`$$NOW` is also available in update pipeline stages:

```javascript
db.sessions.updateMany(
  { active: true },
  [{ $set: { lastSeenAt: "$$NOW" } }]
);
```

## $$CLUSTER_TIME - Cluster Logical Timestamp

`$$CLUSTER_TIME` is the logical cluster time as a Timestamp type, available in replica set and sharded cluster environments. It is used for advanced change stream and oplog operations:

```javascript
db.events.aggregate([
  {
    $project: {
      clusterTimestamp: "$$CLUSTER_TIME"
    }
  }
]);
```

## $$REMOVE - Conditional Field Exclusion

`$$REMOVE` is a special value that, when assigned to a field, causes that field to be excluded from the output document:

```javascript
db.users.aggregate([
  {
    $addFields: {
      sensitiveData: {
        $cond: {
          if: { $eq: ["$role", "admin"] },
          then: "$sensitiveData",
          else: "$$REMOVE"
        }
      }
    }
  }
]);
```

When `role` is not `"admin"`, the `sensitiveData` field is omitted entirely rather than set to null.

## $$DESCEND and $$PRUNE for $redact

These are used with the `$redact` stage to control recursive document traversal:
- `$$DESCEND` - continue recursing into sub-documents
- `$$PRUNE` - exclude the current sub-document and stop recursing
- `$$KEEP` - include the current sub-document and stop recursing

## Summary

`$$ROOT` gives access to the full current document for wrapping or merging. `$$NOW` provides a consistent timestamp across all documents in a pipeline run. `$$REMOVE` omits a field entirely when used as an assignment value in `$addFields`. `$$CURRENT` is the active document node in recursive stages like `$redact`. These system variables eliminate the need for workarounds and make pipelines more expressive.
