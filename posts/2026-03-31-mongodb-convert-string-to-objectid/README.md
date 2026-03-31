# How to Convert String to ObjectId in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, ObjectId, Query, Aggregation, Type Conversion

Description: Learn how to convert a string to an ObjectId in MongoDB using $toObjectId in aggregation, ObjectId() in the shell, and driver-specific methods in application code.

---

MongoDB stores document identifiers as `ObjectId` BSON types, not strings. When a query or aggregation receives a string where an ObjectId is expected, no documents match. Converting strings to ObjectIds is a common requirement when processing user input or migrating data.

## Convert in mongosh

In the MongoDB shell, wrap a 24-character hex string with `ObjectId()`:

```javascript
// Query by converting a string to ObjectId
db.users.findOne({ _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") })
```

## Convert in Aggregation with $toObjectId

Use `$toObjectId` to convert a string field to ObjectId inside an aggregation pipeline:

```javascript
db.orders.aggregate([
  {
    $addFields: {
      customerObjectId: { $toObjectId: "$customerId" }
    }
  },
  {
    $lookup: {
      from: "customers",
      localField: "customerObjectId",
      foreignField: "_id",
      as: "customerDetails"
    }
  }
])
```

This is common when a reference field was stored as a string instead of an ObjectId and you need to join with `$lookup`.

## Convert with $convert for Error Handling

`$convert` provides an `onError` fallback if the string is not a valid ObjectId:

```javascript
db.orders.aggregate([
  {
    $addFields: {
      customerObjectId: {
        $convert: {
          input: "$customerId",
          to: "objectId",
          onError: null,
          onNull: null
        }
      }
    }
  },
  { $match: { customerObjectId: { $ne: null } } }
])
```

## Convert in Application Code

In Node.js with the MongoDB driver:

```javascript
const { ObjectId } = require("mongodb")

const idString = "64f1a2b3c4d5e6f7a8b9c0d1"

// Check validity before converting
if (ObjectId.isValid(idString)) {
  const objectId = new ObjectId(idString)
  const user = await db.collection("users").findOne({ _id: objectId })
}
```

In Python with PyMongo:

```python
from bson import ObjectId

id_string = "64f1a2b3c4d5e6f7a8b9c0d1"

if ObjectId.is_valid(id_string):
    object_id = ObjectId(id_string)
    user = db.users.find_one({"_id": object_id})
```

## Validate Before Converting

Always validate that a string is a valid 24-character hex before attempting conversion to avoid errors:

```javascript
// In mongosh
function isValidObjectId(str) {
  return /^[a-f\d]{24}$/i.test(str)
}

const input = "64f1a2b3c4d5e6f7a8b9c0d1"
if (isValidObjectId(input)) {
  db.users.findOne({ _id: ObjectId(input) })
}
```

## Fix a Collection with Mixed String/ObjectId IDs

If a collection has a mix of string and ObjectId `_id` values, normalize them in a bulk operation:

```javascript
db.orders.find({ _id: { $type: "string" } }).forEach(doc => {
  const newId = ObjectId(doc._id)
  db.orders.insertOne({ ...doc, _id: newId })
  db.orders.deleteOne({ _id: doc._id })
})
```

## Summary

Convert strings to ObjectIds using `ObjectId()` in mongosh, `$toObjectId` or `$convert` in aggregation pipelines, or driver-specific constructors (`new ObjectId(str)` in Node.js, `ObjectId(str)` in Python). Always validate the string format first. Use `$toObjectId` in `$lookup` pipelines when a reference field was accidentally stored as a string.
