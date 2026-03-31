# How to Use $min and $max Operators in MongoDB for Conditional Updates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $min, $max, Update, Operator, Conditional

Description: Learn how to use MongoDB's $min and $max operators to update a field only when the new value is smaller or larger than the current value, enabling safe conditional numeric updates.

---

## How $min and $max Work

`$min` and `$max` are conditional update operators. They compare the update value against the current field value and only apply the change when the condition is satisfied.

- `$min` updates the field only if the new value is **less than** the current value
- `$max` updates the field only if the new value is **greater than** the current value

If the field does not exist, both operators set the field to the specified value.

```mermaid
flowchart LR
    A["Current: {score: 75}"] --> B{$min: {score: 60}}
    B -- "60 < 75 - condition true" --> C["Updated: {score: 60}"]

    D["Current: {score: 75}"] --> E{$min: {score: 90}}
    E -- "90 is not < 75 - no change" --> F["Unchanged: {score: 75}"]
```

## Syntax

```javascript
{ $min: { field: value } }
{ $max: { field: value } }
```

## $min - Keep the Smaller Value

Update only if the new value is less than the current value:

```javascript
// Before: { _id: 1, user: "alice", lowestScore: 75 }

// Attempt to set lowestScore to 60 (60 < 75, update applies)
db.scores.updateOne(
  { _id: 1 },
  { $min: { lowestScore: 60 } }
)
// After: { _id: 1, user: "alice", lowestScore: 60 }

// Attempt to set lowestScore to 90 (90 > 60, no change)
db.scores.updateOne(
  { _id: 1 },
  { $min: { lowestScore: 90 } }
)
// After: { _id: 1, user: "alice", lowestScore: 60 }  (unchanged)
```

## $max - Keep the Larger Value

Update only if the new value is greater than the current value:

```javascript
// Before: { _id: 2, user: "bob", highScore: 80 }

// Attempt to set highScore to 95 (95 > 80, update applies)
db.scores.updateOne(
  { _id: 2 },
  { $max: { highScore: 95 } }
)
// After: { _id: 2, user: "bob", highScore: 95 }

// Attempt to set highScore to 70 (70 < 95, no change)
db.scores.updateOne(
  { _id: 2 },
  { $max: { highScore: 70 } }
)
// After: { _id: 2, user: "bob", highScore: 95 }  (unchanged)
```

## Initializing Non-Existent Fields

When the field does not exist, both `$min` and `$max` set the field to the provided value:

```javascript
// Before: { _id: 3, name: "Carol" }  (no score fields)

db.users.updateOne({ _id: 3 }, { $min: { lowestScore: 75 } })
// After: { _id: 3, name: "Carol", lowestScore: 75 }

db.users.updateOne({ _id: 3 }, { $max: { highScore: 80 } })
// After: { _id: 3, name: "Carol", lowestScore: 75, highScore: 80 }
```

## Using $min for Earliest Date

`$min` works on Date types as well - keep the earliest date:

```javascript
// Before: { _id: 4, user: "dave", firstLoginAt: ISODate("2024-03-15") }

db.users.updateOne(
  { _id: 4 },
  { $min: { firstLoginAt: new Date("2024-01-10") } }
)
// After: firstLoginAt = 2024-01-10 (earlier date wins)

db.users.updateOne(
  { _id: 4 },
  { $min: { firstLoginAt: new Date("2024-12-01") } }
)
// After: firstLoginAt = 2024-01-10 (unchanged, newer date doesn't replace)
```

## Using $max for Latest Date

Keep the most recent date:

```javascript
// Before: { _id: 5, user: "eve", lastLoginAt: ISODate("2024-01-01") }

db.users.updateOne(
  { _id: 5 },
  { $max: { lastLoginAt: new Date() } }
)
// After: lastLoginAt = today's date (newer date wins)
```

## Combining $min and $max in One Update

```javascript
// Track both personal best and personal worst in one operation
db.athletes.updateOne(
  { userId: ObjectId("64a1b2c3d4e5f6789012345a") },
  {
    $min: { personalWorst: 45.2 },
    $max: { personalBest: 45.2 }
  }
)
```

## Practical Use Case - Rate Limiting

Track the earliest request time in a time window:

```javascript
db.rateLimits.updateOne(
  { userId: "user-123", window: "2024-01-15T10:00:00Z" },
  {
    $inc: { requestCount: 1 },
    $min: { firstRequestAt: new Date() },
    $max: { lastRequestAt: new Date() }
  },
  { upsert: true }
)
```

## Use Cases

- Tracking personal best scores (always keep the highest)
- Recording the earliest signup date or first login
- Maintaining a running low-water or high-water mark
- Storing minimum or maximum price seen for a product
- Clamp-style updates where a value should not exceed a bound

## Summary

`$min` and `$max` are conditional update operators that apply an update only when it would make the stored value smaller or larger, respectively. They eliminate the need for a read-check-update pattern for min/max tracking, and they work on numbers, strings, and dates. Both operators initialize a missing field to the provided value. Use them for leaderboards, time tracking, pricing floors and ceilings, and any scenario where you want to preserve an extreme value atomically.
