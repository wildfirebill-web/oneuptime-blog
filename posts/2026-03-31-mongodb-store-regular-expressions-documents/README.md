# How to Store Regular Expressions in MongoDB Documents

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Regex, BSON, Query

Description: Learn how to store BSON regex values in MongoDB documents and use them dynamically in queries, validation rules, and pattern-matching aggregation pipelines.

---

## BSON Regex Type

MongoDB supports a native BSON regex type that stores a regular expression pattern and flags together in a document field. This differs from using a regex to query - you are storing the pattern itself as data, enabling dynamic matching logic driven by database values.

## Storing a Regex in mongosh

```javascript
db.filters.insertOne({
  name: "email-validator",
  pattern: /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/i
});

db.filters.insertOne({
  name: "phone-us",
  pattern: /^\+?1?\d{10,14}$/
});
```

The stored document contains a BSON regex, not a string.

## Reading Stored Regexes

```javascript
const doc = db.filters.findOne({ name: "email-validator" });
print(doc.pattern);       // /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/i
print(typeof doc.pattern); // object
```

## Using Stored Patterns Dynamically in Queries

Retrieve a stored regex and apply it as a filter:

```javascript
const filter = db.filters.findOne({ name: "email-validator" });
const emailRegex = filter.pattern;

// Use the stored regex to filter a collection
db.users.find({ email: emailRegex });
```

This enables configurable validation rules without code deployments.

## Querying for Documents That Contain a Regex Type

```javascript
// Find documents where the 'pattern' field is a regex type (type 11)
db.filters.find({ pattern: { $type: "regex" } });
```

## Storing Regex in Node.js

```javascript
await collection.insertOne({
  name: "slug-validator",
  pattern: /^[a-z0-9-]+$/
});

const doc = await collection.findOne({ name: "slug-validator" });
const regex = doc.pattern;  // native JS RegExp
console.log(regex.test("valid-slug"));   // true
console.log(regex.test("Invalid Slug")); // false
```

## Storing Regex in Python

```python
import re
from bson.regex import Regex

# Store a regex
collection.insert_one({
    "name": "zip-code",
    "pattern": Regex(r"^\d{5}(-\d{4})?$")
})

# Retrieve and use
doc = collection.find_one({"name": "zip-code"})
pattern = re.compile(doc["pattern"].pattern, doc["pattern"].flags)
print(bool(pattern.match("90210")))       # True
print(bool(pattern.match("not-a-zip")))   # False
```

## Using Stored Regex in Aggregation

The `$regexMatch` operator can reference a stored pattern via a variable:

```javascript
const filter = db.filters.findOne({ name: "email-validator" });

db.signups.aggregate([
  {
    $addFields: {
      validEmail: {
        $regexMatch: {
          input: "$email",
          regex: filter.pattern
        }
      }
    }
  },
  { $match: { validEmail: false } }
]);
```

## Summary

MongoDB's BSON regex type lets you store regular expression patterns directly in documents. Retrieve them at runtime to drive dynamic filtering and validation logic. Use `$type: "regex"` to query for documents containing regex fields, and `$regexMatch` in aggregation pipelines to apply stored patterns to other data without hardcoding them in application code.
