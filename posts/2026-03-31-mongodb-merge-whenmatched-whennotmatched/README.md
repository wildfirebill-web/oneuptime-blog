# How to Use $merge with whenMatched and whenNotMatched Options in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $merge, Pipeline, Upsert

Description: Master the whenMatched and whenNotMatched options in MongoDB's $merge stage to control exactly how pipeline results are written to target collections.

---

## Overview of whenMatched and whenNotMatched

The `$merge` stage uses two options to determine what happens during writes:

- `whenMatched`: controls behavior when a document with the same key already exists in the target
- `whenNotMatched`: controls behavior when no matching document exists in the target

Understanding these options is essential for building correct materialized views and incremental aggregations.

## whenMatched Options

### merge (default)

Merges fields from the new document into the existing document. Existing fields not in the new document are preserved:

```javascript
db.sales.aggregate([
  { $group: { _id: "$region", total: { $sum: "$amount" } } },
  {
    $merge: {
      into: "region_stats",
      on: "_id",
      whenMatched: "merge",
      whenNotMatched: "insert"
    }
  }
])
// Existing { _id: "west", total: 500, manager: "Alice" }
// Incoming { _id: "west", total: 750 }
// Result:  { _id: "west", total: 750, manager: "Alice" }
```

### replace

Replaces the entire existing document with the incoming document:

```javascript
{
  $merge: {
    into: "snapshots",
    on: "_id",
    whenMatched: "replace",
    whenNotMatched: "insert"
  }
}
// Existing { _id: "west", total: 500, manager: "Alice" }
// Incoming { _id: "west", total: 750 }
// Result:  { _id: "west", total: 750 }  <-- manager is gone
```

### keepExisting

Keeps the existing document unchanged - effectively skips the update:

```javascript
{
  $merge: {
    into: "first_seen_cache",
    on: "_id",
    whenMatched: "keepExisting",
    whenNotMatched: "insert"
  }
}
// Useful for recording the first occurrence of each key
```

### fail

Throws an error if a duplicate is found - useful when the pipeline should produce unique keys:

```javascript
{
  $merge: {
    into: "unique_events",
    on: "_id",
    whenMatched: "fail",
    whenNotMatched: "insert"
  }
}
```

### Custom Pipeline

Use an update pipeline for fine-grained control with access to both existing (`$$ROOT`) and incoming (`$$new`) document fields:

```javascript
db.daily_events.aggregate([
  { $group: { _id: "$userId", newCount: { $sum: 1 }, newRevenue: { $sum: "$amount" } } },
  {
    $merge: {
      into: "user_lifetime_stats",
      on: "_id",
      whenMatched: [
        {
          $set: {
            totalCount: { $add: ["$totalCount", "$$new.newCount"] },
            totalRevenue: { $add: ["$totalRevenue", "$$new.newRevenue"] },
            lastUpdated: "$$NOW"
          }
        }
      ],
      whenNotMatched: "insert"
    }
  }
])
```

## whenNotMatched Options

### insert (default)

Inserts the incoming document as a new record when no match is found:

```javascript
whenNotMatched: "insert"
```

### discard

Silently ignores the incoming document if no match is found:

```javascript
{
  $merge: {
    into: "known_users_enrichment",
    on: "_id",
    whenMatched: "merge",
    whenNotMatched: "discard"   // Only update existing users, never create new ones
  }
}
```

### fail

Throws an error if no matching document exists:

```javascript
{
  $merge: {
    into: "orders_enriched",
    on: "orderId",
    whenMatched: "merge",
    whenNotMatched: "fail"  // All orders must already exist in target
  }
}
```

## Practical Decision Matrix

```text
Goal                          | whenMatched    | whenNotMatched
------------------------------|----------------|----------------
Upsert (full replace)         | replace        | insert
Partial update existing only  | merge          | discard
Increment counters            | [pipeline]     | insert
First-write wins              | keepExisting   | insert
Enforce no duplicates         | fail           | insert
Update existing, no new rows  | merge          | discard
```

## Summary

The `whenMatched` and `whenNotMatched` options of MongoDB's `$merge` stage precisely control how aggregation results interact with an existing target collection. Use `replace` for full overwrites, `merge` for partial field updates, `keepExisting` for first-write semantics, and a custom pipeline array for arithmetic increments. The `discard` option for `whenNotMatched` restricts `$merge` to updating only existing documents, enabling targeted enrichment without creating new records.
