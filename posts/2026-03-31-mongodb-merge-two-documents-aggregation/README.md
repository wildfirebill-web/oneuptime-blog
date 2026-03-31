# How to Merge Two Documents in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Document

Description: Learn how to merge two documents or objects in MongoDB aggregation using $mergeObjects, $lookup with $mergeObjects, and pipeline-style updates.

---

Merging documents - combining fields from two separate objects into one - is a common aggregation task when joining data from multiple sources, flattening nested structures, or applying default values. MongoDB's `$mergeObjects` operator handles this cleanly.

## $mergeObjects Syntax

`$mergeObjects` takes an array of documents and returns a single document with fields from all inputs. Later fields override earlier ones for duplicate keys:

```javascript
{ $mergeObjects: [<document1>, <document2>, ...] }
```

## Merging Two Fields Within a Document

Combine two embedded sub-documents into one flattened document:

```javascript
db.users.aggregate([
  {
    $project: {
      profile: {
        $mergeObjects: ["$personalInfo", "$contactInfo"]
      }
    }
  }
]);
```

If `personalInfo` is `{ name: "Alice", age: 30 }` and `contactInfo` is `{ email: "alice@example.com", phone: "555-1234" }`, the result is `{ name: "Alice", age: 30, email: "alice@example.com", phone: "555-1234" }`.

## Overriding Fields with $mergeObjects

When the same key appears in multiple documents, the last one wins:

```javascript
db.configs.aggregate([
  {
    $project: {
      effectiveConfig: {
        $mergeObjects: [
          "$defaults",
          "$overrides"
        ]
      }
    }
  }
]);
```

Fields in `$overrides` replace fields in `$defaults`. This is the standard pattern for layered configuration.

## Merging with a Literal Object

Merge a document field with inline literal properties:

```javascript
db.products.aggregate([
  {
    $project: {
      enriched: {
        $mergeObjects: [
          "$productData",
          { source: "inventory-api", fetchedAt: "$$NOW" }
        ]
      }
    }
  }
]);
```

## $mergeObjects as a $group Accumulator

In a `$group` stage, `$mergeObjects` accumulates documents across multiple input documents into one merged object. This is useful for merging partial documents from different sources:

```javascript
db.updates.aggregate([
  {
    $group: {
      _id: "$entityId",
      merged: { $mergeObjects: "$fields" }
    }
  }
]);
```

Each document in the `updates` collection has a partial set of fields. `$mergeObjects` as an accumulator combines all of them into one document per entity, with later updates overriding earlier ones for shared keys.

## Merging Lookup Results

After a `$lookup`, merge the looked-up document into the parent:

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customerData"
    }
  },
  {
    $replaceRoot: {
      newRoot: {
        $mergeObjects: [
          "$$ROOT",
          { $arrayElemAt: ["$customerData", 0] }
        ]
      }
    }
  }
]);
```

This flattens the customer data into the order document, making the result a single merged document.

## Handling Null Values

`$mergeObjects` ignores null and missing documents in the input array:

```javascript
db.products.aggregate([
  {
    $project: {
      data: {
        $mergeObjects: [
          "$baseFields",
          { $ifNull: ["$optionalFields", {}] }
        ]
      }
    }
  }
]);
```

## Summary

Use `$mergeObjects` to combine two or more documents in aggregation. As a project expression it merges document fields; as a group accumulator it merges partial documents across multiple input documents. Later inputs override earlier inputs on key conflicts. It pairs naturally with `$lookup` and `$replaceRoot` for flattening joined data.
