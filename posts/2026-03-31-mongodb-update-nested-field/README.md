# How to Update a Nested Field in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Update, Dot Notation, Operator, Schema

Description: Learn how to update fields inside nested documents in MongoDB using dot notation with $set, avoiding full document replacement of embedded objects.

---

MongoDB documents frequently contain nested (embedded) objects. Updating a single field within an embedded document requires dot notation - using a full document replacement on the nested object would overwrite all sibling fields.

## The Problem with Direct Object Assignment

A common mistake is replacing an entire embedded object when you only want to update one field:

```javascript
// WRONG - this REPLACES the entire address object
db.users.updateOne(
  { _id: userId },
  { $set: { address: { city: "New York" } } }
)
// Result: address now only has "city", losing "street", "zip", "country"
```

## Correct Approach: Dot Notation

Use dot notation to target a specific field within the nested document:

```javascript
// CORRECT - only updates address.city, leaves other address fields intact
db.users.updateOne(
  { _id: userId },
  { $set: { "address.city": "New York" } }
)
```

## Multiple Nested Field Updates

Update multiple nested fields in a single operation:

```javascript
db.users.updateOne(
  { _id: userId },
  {
    $set: {
      "address.city": "New York",
      "address.state": "NY",
      "address.zipCode": "10001"
    }
  }
)
```

## Deeply Nested Fields

Dot notation extends to any depth:

```javascript
db.companies.updateOne(
  { _id: companyId },
  {
    $set: {
      "headquarters.location.country": "USA",
      "headquarters.location.coordinates.lat": 40.7128,
      "headquarters.location.coordinates.lng": -74.0060
    }
  }
)
```

## Updating Nested Fields Conditionally

Combine with a filter on the nested field itself:

```javascript
// Only update if current city is "Boston"
db.users.updateOne(
  {
    _id: userId,
    "address.city": "Boston"
  },
  {
    $set: { "address.city": "Cambridge" }
  }
)
```

## Adding a New Field to an Embedded Document

`$set` with dot notation also creates nested fields that do not exist yet:

```javascript
// Add a new field to the address sub-document
db.users.updateOne(
  { _id: userId },
  { $set: { "address.deliveryInstructions": "Leave at door" } }
)
```

## Removing a Nested Field

Use `$unset` with dot notation to remove a field from an embedded document:

```javascript
db.users.updateOne(
  { _id: userId },
  { $unset: { "address.deliveryInstructions": "" } }
)
```

## Bulk Update of All Documents

Update a nested field across the entire collection:

```javascript
db.users.updateMany(
  { "address.country": { $exists: false } },
  { $set: { "address.country": "US" } }
)
```

## Node.js Example

```javascript
const { ObjectId } = require("mongodb");

async function updateUserCity(userId, newCity) {
  const result = await db.collection("users").updateOne(
    { _id: new ObjectId(userId) },
    { $set: { "address.city": newCity } }
  );

  return result.modifiedCount === 1;
}
```

## Summary

Always use dot notation with `$set` to update individual fields within nested documents. Never replace an entire embedded object with `$set` at the object level unless you intend to overwrite all its fields. Dot notation works to any depth and also creates fields that do not yet exist. Use `$unset` with dot notation to remove nested fields.
