# How to Run JavaScript in mongosh

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongosh, JavaScript, Script

Description: Learn how to use mongosh as a full JavaScript runtime - declare variables, write functions, use async/await, import modules, and iterate cursors programmatically.

---

## mongosh as a JavaScript REPL

`mongosh` is built on the Node.js JavaScript engine, giving you a full ES2022+ environment inside the shell. You can declare variables, write functions, use modern syntax, and interact with MongoDB collections through the `db` global object.

## Variables and Expressions

```javascript
// Declare variables
const dbName = db.getName();
let count = 0;

// String interpolation
print(`Connected to database: ${dbName}`);

// Arithmetic
const avgDocSize = db.stats().avgObjSize;
print(`Average document size: ${(avgDocSize / 1024).toFixed(2)} KB`);
```

## Functions

```javascript
function getCollectionStats(collName) {
  const stats = db.getCollection(collName).stats();
  return {
    collection: collName,
    documents: stats.count,
    sizeMB: (stats.size / 1024 / 1024).toFixed(2)
  };
}

print(JSON.stringify(getCollectionStats("orders"), null, 2));
```

## Iterating Cursors with forEach

```javascript
db.products.find({ inStock: true }).forEach(doc => {
  print(`${doc.name}: $${doc.price}`);
});
```

## Using toArray for Manipulation

```javascript
const results = db.orders.find({ status: "pending" }).toArray();
const total = results.reduce((sum, doc) => sum + doc.amount, 0);
print(`Total pending: $${total.toFixed(2)}`);
```

## Conditional Logic

```javascript
const collections = db.getCollectionNames();

if (collections.includes("archive")) {
  print("Archive collection exists");
} else {
  db.createCollection("archive");
  print("Created archive collection");
}
```

## Loops and Bulk Operations

```javascript
// Insert 10 test documents
for (let i = 0; i < 10; i++) {
  db.test.insertOne({ index: i, value: Math.random(), ts: new Date() });
}

// Batch update with a loop
db.users.find({ tier: "free" }).forEach(user => {
  db.users.updateOne(
    { _id: user._id },
    { $set: { upgradeEligible: user.logins > 50 } }
  );
});
```

## Async/Await in mongosh

mongosh supports top-level `await` because it wraps operations in an async context:

```javascript
// await works at top level
const doc = await db.orders.findOne({ status: "new" });
print(doc._id);

// Promise.all for parallel operations
const [orderCount, userCount] = await Promise.all([
  db.orders.countDocuments(),
  db.users.countDocuments()
]);
print(`Orders: ${orderCount}, Users: ${userCount}`);
```

## Error Handling

```javascript
try {
  db.orders.insertOne({ _id: "duplicate", amount: 100 });
  db.orders.insertOne({ _id: "duplicate", amount: 200 }); // throws
} catch (err) {
  print(`Error code ${err.code}: ${err.message}`);
}
```

## Multi-line Editing

Type `.editor` to open a multi-line editing mode, enter your code, then press `Ctrl+D` to execute.

## Summary

`mongosh` provides a full JavaScript runtime with ES2022+ syntax, top-level `await`, and all Node.js built-in modules accessible via `require()`. Write functions, use loops, handle errors with `try/catch`, and leverage `forEach` and `toArray` to process cursor results. This makes mongosh a powerful scripting environment for database automation.
