# How to Generate and Parse ObjectIds in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, ObjectId, BSON, Driver

Description: Learn how to generate ObjectIds with custom timestamps, parse hex strings into ObjectId instances, and validate ObjectId format across multiple language drivers.

---

## Generating ObjectIds

MongoDB automatically assigns an `ObjectId` to the `_id` field on insert if none is provided. You can also generate one before inserting to know the ID upfront.

```javascript
// mongosh - generate before insert
const newId = new ObjectId();
db.products.insertOne({ _id: newId, name: "Widget", price: 9.99 });
print(newId.toHexString()); // use this ID immediately
```

In Python with PyMongo:

```python
from bson import ObjectId

new_id = ObjectId()
print(new_id)          # 507f1f77bcf86cd799439011
print(str(new_id))     # same as hex string
```

## Generating an ObjectId with a Specific Timestamp

The first 4 bytes of an ObjectId encode a Unix timestamp. You can craft an ObjectId anchored to any point in time:

```javascript
// mongosh - ObjectId from a specific date
function objectIdFromDate(date) {
  const hex = Math.floor(date.getTime() / 1000).toString(16).padStart(8, "0");
  return new ObjectId(hex + "0000000000000000");
}

const anchor = objectIdFromDate(new Date("2025-06-01T00:00:00Z"));
print(anchor);
```

This is useful for time-range queries that use `_id` instead of a separate date field.

## Parsing a Hex String into an ObjectId

```javascript
// mongosh
const hexStr = "507f1f77bcf86cd799439011";
const id = new ObjectId(hexStr);
print(id.getTimestamp()); // ISODate from the embedded timestamp
```

Node.js:

```javascript
const { ObjectId } = require("mongodb");
const id = new ObjectId("507f1f77bcf86cd799439011");
console.log(id.getTimestamp()); // Date object
```

Java (MongoDB Java Driver):

```java
import org.bson.types.ObjectId;

ObjectId id = new ObjectId("507f1f77bcf86cd799439011");
System.out.println(id.getDate()); // Date embedded in ObjectId
```

## Validating ObjectId Format

Before parsing, validate that a string is a valid 24-character hex value:

```javascript
// mongosh
function isValidObjectId(str) {
  return /^[a-f\d]{24}$/i.test(str);
}

print(isValidObjectId("507f1f77bcf86cd799439011")); // true
print(isValidObjectId("not-an-id"));                // false
```

Python:

```python
from bson import ObjectId
from bson.errors import InvalidId

def is_valid_object_id(s):
    try:
        ObjectId(s)
        return True
    except InvalidId:
        return False
```

## Extracting the Counter and Random Components

The ObjectId hex string exposes its components:

```javascript
const id = new ObjectId("507f1f77bcf86cd799439011");
const hex = id.toHexString();

const timestamp = parseInt(hex.slice(0, 8),  16);   // seconds since epoch
const random    = hex.slice(8, 18);                  // 5-byte random value
const counter   = parseInt(hex.slice(18, 24), 16);   // incrementing counter

print({ timestamp: new Date(timestamp * 1000), random, counter });
```

## Summary

Generate ObjectIds before insert when you need the ID immediately, craft timestamp-anchored ObjectIds for time-range filtering on `_id`, and validate hex strings before parsing to avoid errors. All official MongoDB drivers expose `ObjectId` constructors and `getTimestamp()` methods for consistent cross-language handling.
