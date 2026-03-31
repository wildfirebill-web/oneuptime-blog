# How to Use the .mongoshrc.js File for Startup Customization

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongosh, Customization, Startup

Description: Learn how to use the .mongoshrc.js file to customize mongosh on startup with a dynamic prompt, helper functions, connection shortcuts, and auto-run scripts.

---

## What Is .mongoshrc.js

The `.mongoshrc.js` file is a JavaScript file that mongosh executes automatically on every startup (before presenting the prompt). It is the ideal place to define helper functions, customize the prompt, set config options, and create shortcuts for frequently used operations.

Location: `~/.mongoshrc.js` (Linux/macOS) or `%USERPROFILE%\.mongoshrc.js` (Windows).

## Creating the File

```bash
touch ~/.mongoshrc.js
```

Or open it in your editor:

```bash
vim ~/.mongoshrc.js
```

## Dynamic Prompt

The `prompt` variable controls the mongosh prompt. Assign a function for a dynamic prompt:

```javascript
// ~/.mongoshrc.js

prompt = function() {
  const dbName = db.getName();
  try {
    const status = rs.status();
    const stateStr = status.myState === 1 ? "PRIMARY" : "SECONDARY";
    return `[${stateStr}] ${dbName}> `;
  } catch (_) {
    return `${dbName}> `;
  }
};
```

## Helper Functions

Define reusable helpers that are available in every session:

```javascript
// ~/.mongoshrc.js

// Pretty-print collection stats
function stats(collName) {
  const s = db.getCollection(collName).stats();
  print(`Collection: ${collName}`);
  print(`  Documents : ${s.count}`);
  print(`  Size      : ${(s.size / 1024).toFixed(1)} KB`);
  print(`  Indexes   : ${s.nindexes}`);
}

// List slow queries from profiler
function slowQueries(thresholdMs = 100) {
  return db.system.profile
    .find({ millis: { $gt: thresholdMs } })
    .sort({ millis: -1 })
    .limit(10)
    .toArray();
}

// Quickly switch to a database
function sw(name) {
  return db.getSiblingDB(name);
}
```

Use them directly:

```javascript
stats("orders")
slowQueries(200)
const mydb = sw("sales")
```

## Config Settings

Apply config overrides on startup:

```javascript
// ~/.mongoshrc.js

config.set("inspectDepth", Infinity);
config.set("historyLength", 10000);
disableTelemetry();
```

## Greeting Message

```javascript
// ~/.mongoshrc.js

print("Welcome! Connected to: " + db.getMongo().getURI());
print("Helpers available: stats(coll), slowQueries(ms), sw(db)");
```

## Conditional Logic Based on Environment

```javascript
// ~/.mongoshrc.js

const uri = db.getMongo().getURI();
if (uri.includes("prod") || uri.includes("production")) {
  print("WARNING: Connected to PRODUCTION database!");
  prompt = function() { return `[PROD] ${db.getName()}> `; };
} else {
  prompt = function() { return `[dev] ${db.getName()}> `; };
}
```

## Skipping the .mongoshrc.js File

To start mongosh without loading the rc file (useful for clean scripting):

```bash
mongosh --norc "mongodb://localhost:27017"
```

## Summary

The `~/.mongoshrc.js` file runs automatically on each mongosh startup, making it the right place for dynamic prompts, helper functions, config overrides, and environment-aware warnings. Use it to build a personalized shell environment that speeds up daily database work. Skip it for scripts requiring a pristine environment with `--norc`.
