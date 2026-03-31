# How to Convert Data Types in MongoDB Aggregation with $convert

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Operator

Description: Learn how to convert values between BSON types in MongoDB aggregation using $convert and its shorthand operators like $toString and $toInt.

---

Data arriving from different sources often has inconsistent types - prices stored as strings, dates as epoch integers, IDs as both strings and ObjectIds. MongoDB's `$convert` operator handles type conversion in aggregation with explicit control over error and null behavior.

## $convert Syntax

```javascript
{
  $convert: {
    input: <expression>,
    to: <type name or number>,
    onError: <expression>,   // optional
    onNull: <expression>     // optional
  }
}
```

`to` specifies the target type as a string alias or BSON type number. `onError` provides a fallback when conversion fails. `onNull` provides a fallback when `input` is null or missing.

## Basic Type Conversion

Convert a string field to a number:

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      price: {
        $convert: {
          input: "$priceString",
          to: "double",
          onError: 0.0,
          onNull: 0.0
        }
      }
    }
  }
]);
```

## Shorthand Conversion Operators

MongoDB provides shorthand operators for common conversions. These behave like `$convert` but with no `onError`/`onNull` options (they propagate null on null, error on failure):

```javascript
db.products.aggregate([
  {
    $project: {
      priceNum: { $toDouble: "$priceString" },
      idStr:    { $toString: "$_id" },
      ageInt:   { $toInt: "$ageString" },
      ageLong:  { $toLong: "$ageString" },
      score:    { $toDecimal: "$scoreString" },
      active:   { $toBool: "$activeString" },
      created:  { $toDate: "$createdEpochMs" },
      oid:      { $toObjectId: "$idString" }
    }
  }
]);
```

Use the shorthand when you are confident the input is valid and want concise expressions.

## Converting Epoch Milliseconds to Date

```javascript
db.events.aggregate([
  {
    $project: {
      eventDate: {
        $convert: {
          input: "$timestampMs",
          to: "date",
          onError: null,
          onNull: null
        }
      }
    }
  }
]);
```

Equivalent shorthand: `{ $toDate: "$timestampMs" }`.

## Converting ObjectId to String (and Back)

ObjectIds contain a timestamp, but comparing across documents requires string normalization:

```javascript
db.logs.aggregate([
  {
    $project: {
      idString: { $toString: "$_id" },
      idTimestamp: {
        $toDate: "$_id"  // extract embedded timestamp from ObjectId
      }
    }
  }
]);
```

Convert a string back to ObjectId for lookups:

```javascript
db.references.aggregate([
  {
    $addFields: {
      userId: { $toObjectId: "$userIdString" }
    }
  },
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "user"
    }
  }
]);
```

## Handling Conversion Errors

When input may be invalid (non-numeric strings, malformed dates), use `onError` to substitute a safe default rather than failing the pipeline:

```javascript
db.imports.aggregate([
  {
    $project: {
      quantity: {
        $convert: {
          input: "$rawQuantity",
          to: "int",
          onError: -1,     // sentinel for invalid
          onNull: 0        // default for missing
        }
      }
    }
  }
]);
```

## Batch-Migrating Types in a Collection

Use an update pipeline to permanently convert stored types:

```javascript
db.products.updateMany(
  { priceString: { $exists: true } },
  [
    {
      $set: {
        price: {
          $convert: {
            input: "$priceString",
            to: "double",
            onError: null,
            onNull: null
          }
        }
      }
    },
    { $unset: "priceString" }
  ]
);
```

## Summary

Use `$convert` when you need explicit control over `onError` and `onNull` behavior during type conversion. Use shorthand operators like `$toInt`, `$toString`, `$toDate`, and `$toObjectId` for clean, concise expressions when input is reliably valid. Combine with update pipelines to permanently migrate stored field types.
