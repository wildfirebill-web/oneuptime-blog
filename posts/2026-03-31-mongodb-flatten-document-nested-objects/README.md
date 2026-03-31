# How to Flatten a Document with Nested Objects in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Schema, Transformation

Description: Learn how to flatten nested sub-documents in MongoDB using $replaceRoot, $mergeObjects, and $project in the aggregation pipeline.

---

## What Flattening Means

"Flattening" a document means taking nested sub-documents and promoting their fields to the top level. For example:

```json
{
  "_id": 1,
  "name": "Alice",
  "address": {
    "city": "London",
    "postcode": "EC1A 1BB"
  }
}
```

After flattening becomes:

```json
{
  "_id": 1,
  "name": "Alice",
  "city": "London",
  "postcode": "EC1A 1BB"
}
```

This is useful when exporting data to flat file formats or when querying tools expect top-level fields.

## Flattening One Level with $replaceRoot and $mergeObjects

`$mergeObjects` combines multiple documents into one. Pair it with `$replaceRoot` to replace the document root:

```javascript
db.users.aggregate([
  {
    $replaceRoot: {
      newRoot: {
        $mergeObjects: ["$$ROOT", "$address"]
      }
    }
  },
  {
    $unset: "address"   // remove the now-redundant nested field
  }
])
```

`$$ROOT` represents the current document. `$mergeObjects` merges them left to right, so `address` fields overwrite any top-level conflicts.

## Flattening Multiple Nested Objects

If a document has several nested sub-documents:

```javascript
db.orders.aggregate([
  {
    $replaceRoot: {
      newRoot: {
        $mergeObjects: [
          "$$ROOT",
          "$customer",
          "$shipping"
        ]
      }
    }
  },
  {
    $unset: ["customer", "shipping"]
  }
])
```

## Selective Flattening with $project

When you want to flatten only some fields and rename others:

```javascript
db.users.aggregate([
  {
    $project: {
      name: 1,
      city:     "$address.city",
      postcode: "$address.postcode",
      country:  "$address.country"
    }
  }
])
```

Dot notation lets you reference nested fields directly in `$project`.

## Flattening Arrays of Sub-Documents with $unwind

If the nested field is an array, use `$unwind` first:

```javascript
db.orders.aggregate([
  { $unwind: "$items" },
  {
    $replaceRoot: {
      newRoot: {
        $mergeObjects: ["$$ROOT", "$items"]
      }
    }
  },
  { $unset: "items" }
])
```

This produces one document per array element, each with the item fields promoted to the top level.

## Handling Missing Nested Fields

If some documents lack the nested field, `$mergeObjects` with a missing field is safe - it simply skips the missing value. To be explicit, use `$ifNull`:

```javascript
db.users.aggregate([
  {
    $replaceRoot: {
      newRoot: {
        $mergeObjects: ["$$ROOT", { $ifNull: ["$address", {}] }]
      }
    }
  }
])
```

## Python Example

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017")
db = client["myapp"]

pipeline = [
    {
        "$replaceRoot": {
            "newRoot": {
                "$mergeObjects": ["$$ROOT", {"$ifNull": ["$address", {}]}]
            }
        }
    },
    {"$unset": "address"}
]

for doc in db.users.aggregate(pipeline):
    print(doc)
```

## Summary

Use `$mergeObjects` with `$replaceRoot` to promote a nested sub-document's fields to the top level. Pass multiple sub-documents to `$mergeObjects` to flatten several nested objects at once. Use `$unset` to remove the now-redundant nested field from the output. For array-of-objects flattening, combine `$unwind` with `$replaceRoot`. Use `$ifNull` to guard against documents that lack the nested field.
