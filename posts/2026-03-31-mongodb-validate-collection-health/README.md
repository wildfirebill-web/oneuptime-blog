# How to Use the validate Command to Check Collection Health in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Validate, Collection, Health

Description: Learn how to use MongoDB's validate command to check collection integrity, detect corruption, verify index consistency, and interpret validation output.

---

## What the validate Command Does

The `validate` command scans a collection's data files and indexes to check for internal inconsistencies, document corruption, and broken index entries. It is a diagnostic tool - it does not repair data, only reports problems.

```javascript
// Basic validation
db.runCommand({ validate: "orders" });

// Or using the shell helper
db.orders.validate();
```

## Full vs. Normal Validation

By default, validation performs a normal scan. Full validation reads every data page from disk, providing more thorough detection at the cost of performance:

```javascript
// Full validation (slower, more thorough)
db.runCommand({ validate: "orders", full: true });

// Background validation (MongoDB 5.0+) - less impact on performance
db.runCommand({ validate: "orders", background: true });
```

## Interpreting the Output

A healthy collection returns:

```javascript
{
  ns: "mydb.orders",
  nrecords: 15000,
  nIndexes: 3,
  keysPerIndex: { "_id_": 15000, "status_1": 15000, "customerId_1": 15000 },
  indexDetails: { "_id_": { valid: true }, "status_1": { valid: true }, ... },
  valid: true,
  errors: [],
  warnings: [],
  ok: 1
}
```

A collection with issues returns `valid: false` and populates the `errors` array.

## Checking Specific Validation Fields

```javascript
const result = db.orders.validate({ full: true });

if (!result.valid) {
  print("Collection has issues:");
  result.errors.forEach(err => print("  ERROR: " + err));
  result.warnings.forEach(w => print("  WARN: " + w));
} else {
  print(`Collection is valid. ${result.nrecords} documents, ${result.nIndexes} indexes`);
}
```

## Validating All Collections in a Database

```javascript
db.getCollectionNames().forEach(collName => {
  const result = db.runCommand({ validate: collName });
  if (!result.valid) {
    print(`INVALID: ${collName}`);
    printjson(result.errors);
  } else {
    print(`OK: ${collName}`);
  }
});
```

## When to Run validate

Run validation after:

- Unclean server shutdowns
- Storage hardware failures or RAID rebuilds
- Mongod crashes or signal 11 segfaults
- Unexpected data inconsistencies discovered in application logic
- Before and after major migrations

Avoid running full validation on large collections during peak traffic - it can cause significant I/O load.

## Repair Options

If `validate` finds corruption, the next steps depend on the deployment:

```javascript
// Compact the collection to reclaim space and rebuild data files
db.runCommand({ compact: "orders" });

// For index corruption, rebuild all indexes
db.orders.reIndex();

// For severe corruption, restore from backup
// mongorestore --db mydb --collection orders dump/mydb/orders.bson
```

## validate vs. repairDatabase

`repairDatabase` rebuilds all collections and indexes in a database. It is a last resort for severe corruption and requires significant disk space for the rebuild:

```javascript
// Use only for severe corruption - requires downtime
db.repairDatabase();
```

## Summary

Use MongoDB's `validate` command to check collection and index integrity after abnormal events. Run `{ background: true }` on large production collections to minimize performance impact. Check `result.valid` and the `errors` array to identify issues, and follow up with `reIndex()` for index corruption or a backup restore for data corruption. Validate all collections periodically as part of operational health checks.
