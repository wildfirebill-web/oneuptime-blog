# How to Use ISODate() in MongoDB Shell

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Date, Shell

Description: Learn how to use ISODate() in the MongoDB shell to insert, query, and manipulate date values with precision and clarity.

---

The MongoDB shell provides `ISODate()` as a helper function to create BSON Date objects from ISO 8601 strings. It is the most readable and reliable way to work with dates interactively in the shell or in scripts.

## What ISODate() Does

`ISODate()` parses an ISO 8601 string and returns a BSON Date object stored as UTC milliseconds. The two shell helpers `ISODate()` and `new Date()` are functionally equivalent when given an ISO string, but `ISODate()` is preferred for clarity:

```javascript
ISODate("2026-03-31T12:00:00Z")
// Returns: ISODate("2026-03-31T12:00:00.000Z")
```

Without arguments, `ISODate()` returns the current UTC time - the same as `new Date()`:

```javascript
ISODate()
// Returns: ISODate("2026-03-31T14:23:11.456Z")
```

## Inserting Documents with ISODate()

```javascript
db.orders.insertOne({
  orderId: "ORD-001",
  placedAt: ISODate("2026-03-31T09:00:00Z"),
  deliveredAt: ISODate("2026-04-02T17:30:00Z")
});
```

The stored value is a proper BSON Date, not a string. This distinction matters: string-stored dates sort lexicographically, not chronologically, and cannot use date-specific aggregation operators.

## Querying with ISODate()

Use `ISODate()` in queries just as you would use `new Date()`:

```javascript
// Find orders placed today
db.orders.find({
  placedAt: {
    $gte: ISODate("2026-03-31T00:00:00Z"),
    $lt:  ISODate("2026-04-01T00:00:00Z")
  }
});
```

For relative date queries, compute the boundary in the shell:

```javascript
// Orders placed in the last 7 days
const sevenDaysAgo = new Date();
sevenDaysAgo.setDate(sevenDaysAgo.getDate() - 7);

db.orders.find({
  placedAt: { $gte: sevenDaysAgo }
});
```

## Supported ISO 8601 Formats

`ISODate()` accepts several formats:

```text
"2026-03-31"                  -> assumes midnight UTC
"2026-03-31T12:00:00"         -> local time (no offset - avoid this)
"2026-03-31T12:00:00Z"        -> explicitly UTC (preferred)
"2026-03-31T12:00:00.000Z"    -> with milliseconds
"2026-03-31T12:00:00+05:30"   -> with timezone offset (stored as UTC)
```

Always use the `Z` suffix or an explicit offset to avoid ambiguity.

## Extracting Date Components

Once a date is stored, you can extract its parts using aggregation operators:

```javascript
db.orders.aggregate([
  {
    $project: {
      year:  { $year: "$placedAt" },
      month: { $month: "$placedAt" },
      day:   { $dayOfMonth: "$placedAt" },
      hour:  { $hour: "$placedAt" }
    }
  }
]);
```

These operators operate on the UTC value. If you need local components, pass a `timezone` argument:

```javascript
db.orders.aggregate([
  {
    $project: {
      localParts: {
        $dateToParts: {
          date: "$placedAt",
          timezone: "America/Chicago"
        }
      }
    }
  }
]);
```

## Comparing Two Date Fields

You can compare two date fields within the same document using `$expr`:

```javascript
db.orders.find({
  $expr: { $gt: ["$deliveredAt", "$placedAt"] }
});
```

This returns orders where the delivery time is after placement time - which should always be true, but checking catches data quality issues.

## Summary

`ISODate()` is the canonical way to work with dates in the MongoDB shell. Always provide an ISO 8601 string with a UTC offset to ensure consistent storage. Use BSON Date types rather than strings to enable date arithmetic, aggregation operators, and correct index-based sorting.
