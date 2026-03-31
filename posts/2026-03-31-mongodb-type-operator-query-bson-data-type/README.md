# How to Use $type to Query by BSON Data Type in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query Operator, $type, BSON, Data Type

Description: Learn how to use MongoDB's $type operator to query documents based on the BSON type of a field, using both numeric codes and string aliases.

---

## What Is $type

The `$type` operator matches documents where a field's value has a specific BSON type. MongoDB stores data in BSON format, which supports more data types than JSON. Use `$type` to detect type inconsistencies, validate schemas, or filter by type.

Syntax:

```javascript
{ field: { $type: <BSON type number or alias> } }
```

## Common BSON Type Aliases

MongoDB accepts string aliases (preferred) or numeric type codes:

| Alias | Type | Number |
|-------|------|--------|
| `"double"` | 64-bit float | 1 |
| `"string"` | UTF-8 string | 2 |
| `"object"` | Embedded document | 3 |
| `"array"` | Array | 4 |
| `"bool"` | Boolean | 8 |
| `"date"` | Date | 9 |
| `"null"` | Null | 10 |
| `"int"` | 32-bit integer | 16 |
| `"long"` | 64-bit integer | 18 |
| `"decimal"` | 128-bit decimal | 19 |
| `"objectId"` | ObjectId | 7 |

## Basic Examples

```javascript
// Find documents where age is stored as a string (data quality issue)
db.users.find({ age: { $type: "string" } })

// Find documents where price is a number (either int, long, or double)
db.products.find({ price: { $type: "number" } })
```

The `"number"` alias matches all numeric types (double, int, long, decimal).

## Detecting Mixed Type Fields

Mixed-type fields are a common cause of query bugs. Use `$type` to find them:

```javascript
// Find documents where userId is stored as ObjectId (should be consistent)
const objectIdUsers = await db.collection("orders").countDocuments({
  userId: { $type: "objectId" }
});

// Find documents where userId is stored as string (inconsistency)
const stringUsers = await db.collection("orders").countDocuments({
  userId: { $type: "string" }
});

console.log(`ObjectId: ${objectIdUsers}, String: ${stringUsers}`);
```

## Querying for Array Fields

```javascript
// Find documents where tags is an array
db.posts.find({ tags: { $type: "array" } })

// Find documents where tags is a string (not an array)
db.posts.find({ tags: { $type: "string" } })
```

## Matching Multiple Types

Pass an array of types to match any of them:

```javascript
// Match documents where value is either int or double
db.measurements.find({
  value: { $type: ["int", "double"] }
})
```

## Data Cleanup with $type

Fix type inconsistencies identified with `$type`:

```javascript
// Find all documents where price is stored as string
const badPriceDocs = await db.collection("products")
  .find({ price: { $type: "string" } })
  .toArray();

// Convert string prices to numbers
for (const doc of badPriceDocs) {
  await db.collection("products").updateOne(
    { _id: doc._id },
    { $set: { price: parseFloat(doc.price) } }
  );
}
```

## $type in Aggregation

Use `$type` as an aggregation expression to return the type of a field:

```javascript
db.orders.aggregate([
  {
    $project: {
      fieldType: { $type: "$userId" }
    }
  }
])
```

This returns the type name (e.g., `"string"`, `"objectId"`) as a string field.

## Common Mistakes

- Confusing `$type` (query operator) with `$type` (aggregation expression) - they differ in syntax.
- Using numeric type codes instead of string aliases - aliases are more readable and version-stable.
- Forgetting that `$type: "number"` matches double, int, long, and decimal.

## Summary

The `$type` operator filters MongoDB documents by the BSON type of a field, using readable string aliases like `"string"`, `"int"`, and `"date"`. Use it to detect type inconsistencies in mixed-type fields, audit schema quality, and clean up migration artifacts. In aggregation pipelines, `$type` is also an expression that returns the type name of a field's value.
