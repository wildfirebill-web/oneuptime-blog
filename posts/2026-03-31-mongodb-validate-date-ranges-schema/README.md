# How to Validate Date Ranges in MongoDB Schema Validation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema Validation, Date, Range, Data Quality

Description: Learn how to validate date ranges in MongoDB using $jsonSchema with bsonType date constraints and $expr for cross-field start/end date relationship validation.

---

## Date Type Validation with $jsonSchema

The simplest date validation ensures a field contains a valid BSON date rather than a string or number:

```javascript
db.createCollection("events", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["startDate", "endDate"],
      properties: {
        startDate: {
          bsonType: "date",
          description: "Event start date (BSON Date required)"
        },
        endDate: {
          bsonType: "date",
          description: "Event end date (BSON Date required)"
        }
      }
    }
  }
});
```

This rejects string dates like `"2026-04-01"` - only proper `ISODate()` / `new Date()` values pass.

## Cross-Field Validation: Start Must Precede End

`$jsonSchema` alone cannot compare two fields. Use `$expr` alongside `$jsonSchema` for cross-field constraints:

```javascript
db.createCollection("bookings", {
  validator: {
    $and: [
      {
        $jsonSchema: {
          bsonType: "object",
          required: ["startDate", "endDate"],
          properties: {
            startDate: { bsonType: "date" },
            endDate:   { bsonType: "date" }
          }
        }
      },
      {
        $expr: {
          $lt: ["$startDate", "$endDate"]
        }
      }
    ]
  },
  validationAction: "error",
  validationLevel: "strict"
});
```

## Testing Cross-Field Validation

```javascript
// Valid - end is after start
db.bookings.insertOne({
  startDate: ISODate("2026-04-01T10:00:00Z"),
  endDate:   ISODate("2026-04-01T11:00:00Z")
});

// Invalid - end before start
db.bookings.insertOne({
  startDate: ISODate("2026-04-01T11:00:00Z"),
  endDate:   ISODate("2026-04-01T10:00:00Z")
});
// MongoServerError: Document failed validation

// Invalid - same time (not strictly less than)
db.bookings.insertOne({
  startDate: ISODate("2026-04-01T10:00:00Z"),
  endDate:   ISODate("2026-04-01T10:00:00Z")
});
// MongoServerError: Document failed validation
```

To allow same-time (zero-duration) events, change `$lt` to `$lte`.

## Bounding Dates to Acceptable Ranges

Restrict dates to a reasonable range to catch obviously wrong values like year 1970 or year 9999:

```javascript
{
  $and: [
    {
      $jsonSchema: {
        bsonType: "object",
        properties: {
          startDate: { bsonType: "date" },
          endDate:   { bsonType: "date" }
        }
      }
    },
    {
      $expr: {
        $and: [
          { $gte: ["$startDate", ISODate("2020-01-01T00:00:00Z")] },
          { $lte: ["$endDate",   ISODate("2030-12-31T23:59:59Z")] },
          { $lt:  ["$startDate", "$endDate"] }
        ]
      }
    }
  ]
}
```

## Validating Duration Constraints

Enforce minimum and maximum duration using `$subtract` on dates:

```javascript
{
  $expr: {
    $and: [
      { $lt: ["$startDate", "$endDate"] },
      // Duration must be at least 15 minutes (900000 ms)
      { $gte: [{ $subtract: ["$endDate", "$startDate"] }, 900000] },
      // Duration must not exceed 30 days (2592000000 ms)
      { $lte: [{ $subtract: ["$endDate", "$startDate"] }, 2592000000] }
    ]
  }
}
```

## Optional Date Fields with Conditional Validation

Some schemas have optional date ranges that must be valid only when present:

```javascript
{
  $jsonSchema: {
    bsonType: "object",
    properties: {
      promoStart: { bsonType: ["date", "null"] },
      promoEnd:   { bsonType: ["date", "null"] }
    }
  }
}
```

Combine with `$expr` and null checks:

```javascript
{
  $expr: {
    $or: [
      { $eq: ["$promoStart", null] },
      { $eq: ["$promoEnd",   null] },
      { $lt: ["$promoStart", "$promoEnd"] }
    ]
  }
}
```

## Summary

Validating date ranges in MongoDB requires two layers: `$jsonSchema` with `bsonType: "date"` to enforce correct data types, and `$expr` with `$lt` or `$lte` for cross-field comparison of start and end dates. Combine both in a `$and` validator for complete coverage. Use `$subtract` in `$expr` to enforce minimum and maximum duration constraints. This approach catches type errors, invalid date ordering, and out-of-range values at the database level.
