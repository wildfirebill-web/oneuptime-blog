# How to Use $[identifier] with arrayFilters to Update Specific Array Elements

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array Operator, Update Operator, Aggregation

Description: Learn how to use the $[identifier] positional operator with arrayFilters to selectively update array elements that match a condition in MongoDB documents.

---

## What Is the $[identifier] Operator?

The `$[identifier]` filtered positional operator, used together with the `arrayFilters` option, allows you to update only specific elements of an array that satisfy a given condition. This is the most precise array update mechanism in MongoDB, supporting both flat arrays and nested arrays of objects.

## Basic Syntax

```javascript
db.collection.updateOne(
  { <filter> },
  { $set: { "arrayField.$[elem].<field>": <newValue> } },
  { arrayFilters: [{ "elem.<condition>": <value> }] }
)
```

The `elem` identifier (any name you choose) links the update path to the filter condition.

## Example: Updating Specific Array Elements by Condition

```javascript
// Document:
// { _id: 1, scores: [{ subject: "math", score: 70 }, { subject: "science", score: 80 }] }

db.students.updateOne(
  { _id: 1 },
  { $set: { "scores.$[s].score": 90 } },
  { arrayFilters: [{ "s.subject": "math" }] }
)
// Updates the math score to 90 without affecting the science score
```

## Example: Marking Tasks as Complete

```javascript
db.projects.updateOne(
  { _id: "project-1" },
  { $set: { "tasks.$[t].status": "done" } },
  { arrayFilters: [{ "t.taskId": "task-42" }] }
)
```

## Updating Multiple Matching Elements

`updateOne` updates all matching elements within a single document. Use `updateMany` to affect multiple documents.

```javascript
db.orders.updateMany(
  {},
  { $set: { "items.$[item].discount": 0.1 } },
  { arrayFilters: [{ "item.category": "electronics", "item.price": { $gt: 100 } }] }
)
```

All items in all orders where category is electronics and price exceeds 100 get a 10% discount.

## Nested Array Updates

You can chain identifiers for nested arrays.

```javascript
db.courses.updateOne(
  { _id: "course-1" },
  { $set: { "modules.$[m].lessons.$[l].completed": true } },
  {
    arrayFilters: [
      { "m.moduleId": "mod-2" },
      { "l.lessonId": "lesson-5" }
    ]
  }
)
```

## Incrementing a Field in Matching Elements

```javascript
db.inventory.updateMany(
  {},
  { $inc: { "items.$[item].stock": -1 } },
  { arrayFilters: [{ "item.sku": "WIDGET-A", "item.stock": { $gt: 0 } }] }
)
```

## Multiple Conditions in a Single Filter

```javascript
db.employees.updateMany(
  {},
  { $set: { "shifts.$[s].bonus": 50 } },
  {
    arrayFilters: [
      {
        "s.hours": { $gte: 8 },
        "s.date": { $gte: new Date("2024-01-01") }
      }
    ]
  }
)
```

## $[identifier] vs Positional Operator ($)

| Operator | Matches | Supports Conditions |
|----------|---------|---------------------|
| `$` | First matching element | Limited to query filter |
| `$[]` | All elements | None |
| `$[identifier]` | Elements matching arrayFilters | Full query expression |

## Summary

The `$[identifier]` operator combined with `arrayFilters` is the most powerful array update tool in MongoDB. It allows surgical updates to array elements matching complex conditions, supports nested arrays through chained identifiers, and works with any update operator including `$set`, `$inc`, `$unset`, and more.
