# How to Use $type to Query by BSON Data Type in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $type, BSON, Data Types, Query Operators, Schema Validation

Description: Learn how to use MongoDB's $type operator to query documents by the BSON data type of a field, useful for data quality checks and mixed-type field handling.

---

## What Is the $type Operator?

The `$type` operator selects documents where the value of a field matches a specified BSON type. MongoDB is schema-flexible, meaning the same field can contain different types across documents. `$type` lets you query and identify those variations.

```javascript
// Find documents where 'age' is stored as a number
db.users.find({ age: { $type: 'number' } })

// Find documents where 'price' is stored as a string (data quality issue)
db.products.find({ price: { $type: 'string' } })
```

## BSON Type Reference

You can specify types by name (string) or by number (BSON type code):

| Type | String Alias | Number |
|------|-------------|--------|
| Double | `"double"` | 1 |
| String | `"string"` | 2 |
| Object | `"object"` | 3 |
| Array | `"array"` | 4 |
| Boolean | `"bool"` | 8 |
| Date | `"date"` | 9 |
| Null | `"null"` | 10 |
| Regular Expression | `"regex"` | 11 |
| 32-bit Integer | `"int"` | 16 |
| Timestamp | `"timestamp"` | 17 |
| 64-bit Integer | `"long"` | 18 |
| Decimal128 | `"decimal"` | 19 |
| ObjectId | `"objectId"` | 7 |

## Basic Usage

```javascript
const { MongoClient } = require('mongodb');
const client = new MongoClient('mongodb://localhost:27017');
const db = client.db('myapp');
const products = db.collection('products');

// Find products where 'price' is stored as a number
const correctPrices = await products.find({
  price: { $type: 'double' }
}).toArray();

// Find products where 'price' is stored as a string (bad data)
const wrongPrices = await products.find({
  price: { $type: 'string' }
}).toArray();

// Find documents where a field is an array
const withTagsArray = await products.find({
  tags: { $type: 'array' }
}).toArray();

// Find documents where a field is null
const withNullStatus = await products.find({
  status: { $type: 'null' }
}).toArray();
```

## Using "number" Alias for Any Numeric Type

The `"number"` alias matches int, long, double, and decimal:

```javascript
// Matches int32, int64, double, decimal128
const numericPrices = await products.find({
  price: { $type: 'number' }
}).toArray();
```

## Matching Multiple Types

Pass an array of types to match any of them:

```javascript
// Find documents where 'value' is either a number or a string
const mixed = await products.find({
  value: { $type: ['number', 'string'] }
}).toArray();

// Find fields that store date OR timestamp type
const dateLike = await db.collection('events').find({
  startTime: { $type: ['date', 'timestamp'] }
}).toArray();
```

## Data Quality Audit

Use `$type` to find type inconsistencies in your collection:

```javascript
async function auditFieldTypes(collection, fieldName, expectedType) {
  const wrongType = await collection.find({
    [fieldName]: { $exists: true, $not: { $type: expectedType } }
  }).toArray();

  console.log(`Found ${wrongType.length} document(s) where '${fieldName}' is not ${expectedType}`);
  return wrongType;
}

// Check for price fields that are not numbers
const badPrices = await auditFieldTypes(products, 'price', 'number');

// Fix the bad prices
for (const doc of badPrices) {
  const numericPrice = parseFloat(doc.price);
  if (!isNaN(numericPrice)) {
    await products.updateOne(
      { _id: doc._id },
      { $set: { price: numericPrice } }
    );
  }
}
```

## Using $type in Aggregation

```javascript
// Count documents by field type
const typeCounts = await db.collection('products').aggregate([
  {
    $group: {
      _id: { $type: '$price' },
      count: { $sum: 1 },
    }
  }
]).toArray();

// Output: [{ _id: 'double', count: 850 }, { _id: 'string', count: 3 }]
```

## PyMongo Usage

```python
from pymongo import MongoClient

client = MongoClient('mongodb://localhost:27017')
db = client['myapp']
products = db['products']

# Find products with numeric price
correct = list(products.find({'price': {'$type': 'number'}}))

# Find products with string price (bad data)
wrong = list(products.find({'price': {'$type': 'string'}}))

# Find products where tags is an array
with_tags = list(products.find({'tags': {'$type': 'array'}}))
```

## $type in Schema Validation

Combine `$type` with `$jsonSchema` for runtime validation:

```javascript
await db.createCollection('products', {
  validator: {
    $jsonSchema: {
      bsonType: 'object',
      required: ['name', 'price', 'category'],
      properties: {
        price: { bsonType: 'number', description: 'must be a number' },
        name: { bsonType: 'string' },
        category: { bsonType: 'string' },
      }
    }
  }
});
```

## Summary

MongoDB's `$type` operator queries documents by the BSON data type of a field, which is invaluable for data quality audits in flexible-schema collections. Use string aliases like `'string'`, `'number'`, `'array'`, and `'date'` for readability. The `'number'` alias conveniently matches all numeric types (int, long, double, decimal). In aggregation, use `$type` as an expression operator to classify documents by field type, enabling type distribution analysis across your collection.
