# How to Conditionally Add Fields in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline, Expression

Description: Learn how to conditionally add or compute fields in MongoDB aggregation pipelines using $addFields, $cond, $switch, and $ifNull with practical examples.

---

## The $addFields Stage

`$addFields` (aliased as `$set` in newer MongoDB versions) appends new fields to each document without removing existing ones. Combine it with conditional expressions to add fields only when certain conditions are met.

## Basic Conditional with $cond

`$cond` is MongoDB's ternary operator - it takes an `if` condition, a `then` value, and an `else` value:

```javascript
db.orders.aggregate([
  {
    $addFields: {
      label: {
        $cond: {
          if: { $gte: ["$total", 100] },
          then: "large",
          else: "small"
        }
      }
    }
  }
])
```

Shorthand array form also works:

```javascript
{ $cond: [ { $gte: ["$total", 100] }, "large", "small" ] }
```

## Multi-Branch Logic with $switch

`$switch` is the equivalent of a `case` statement:

```javascript
db.products.aggregate([
  {
    $addFields: {
      priceCategory: {
        $switch: {
          branches: [
            { case: { $lt:  ["$price", 10]  }, then: "budget"   },
            { case: { $lt:  ["$price", 50]  }, then: "mid-range" },
            { case: { $gte: ["$price", 50]  }, then: "premium"  }
          ],
          default: "unknown"
        }
      }
    }
  }
])
```

Always include a `default` to handle documents that match no branch.

## Adding a Field Only When Another Field Exists

Use `$cond` with `$ifNull` to add a field only when a source field is present:

```javascript
db.users.aggregate([
  {
    $addFields: {
      displayName: {
        $cond: {
          if: { $gt: [{ $ifNull: ["$nickname", null] }, null] },
          then: "$nickname",
          else: "$username"
        }
      }
    }
  }
])
```

Or more concisely with `$ifNull` alone when you want a fallback:

```javascript
db.users.aggregate([
  {
    $addFields: {
      displayName: { $ifNull: ["$nickname", "$username"] }
    }
  }
])
```

## Adding a Computed Boolean Flag

Mark documents as overdue based on a date comparison:

```javascript
db.invoices.aggregate([
  {
    $addFields: {
      isOverdue: {
        $and: [
          { $lt: ["$dueDate", new Date()] },
          { $ne: ["$status", "paid"] }
        ]
      }
    }
  }
])
```

## Combining Multiple Conditional Fields

Add several computed fields in a single `$addFields` stage:

```javascript
db.employees.aggregate([
  {
    $addFields: {
      annualSalary: { $multiply: ["$monthlySalary", 12] },
      seniorityLevel: {
        $switch: {
          branches: [
            { case: { $lt:  ["$yearsExperience", 2] }, then: "junior"  },
            { case: { $lt:  ["$yearsExperience", 5] }, then: "mid"     },
            { case: { $gte: ["$yearsExperience", 5] }, then: "senior"  }
          ],
          default: "unknown"
        }
      },
      isFullTime: { $eq: ["$employmentType", "full-time"] }
    }
  }
])
```

## Python Example

```python
from pymongo import MongoClient
from datetime import datetime

client = MongoClient("mongodb://localhost:27017")
db = client["hr"]

pipeline = [
    {
        "$addFields": {
            "annualSalary": {"$multiply": ["$monthlySalary", 12]},
            "isOverdue": {
                "$and": [
                    {"$lt": ["$reviewDate", datetime.utcnow()]},
                    {"$ne": ["$reviewed", True]}
                ]
            }
        }
    }
]

for doc in db.employees.aggregate(pipeline):
    print(doc["name"], doc.get("annualSalary"), doc.get("isOverdue"))
```

## Summary

Use `$addFields` (or `$set`) to append computed fields without dropping existing data. Use `$cond` for binary if/else logic and `$switch` for multi-branch conditions. Use `$ifNull` to provide fallback values when a field may be absent. You can add multiple computed fields in a single stage, keeping the pipeline concise and readable.
