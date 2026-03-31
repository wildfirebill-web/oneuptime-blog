# How to Use Switch-Case Logic in MongoDB Aggregation with $switch

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Operator

Description: Learn how to use $switch in MongoDB aggregation to evaluate multiple conditions and return the first matching branch value.

---

When conditional logic requires more than two branches, nesting multiple `$cond` operators becomes hard to read. `$switch` provides a clean multi-branch alternative, evaluating conditions in order and returning the first match.

## $switch Syntax

```javascript
{
  $switch: {
    branches: [
      { case: <expression1>, then: <value1> },
      { case: <expression2>, then: <value2> },
      ...
    ],
    default: <default value>
  }
}
```

Each branch has a `case` (boolean expression) and a `then` (return value). The first `case` that evaluates to `true` determines the output. `default` handles the case where no branch matches.

## Basic Grading Example

Assign letter grades from a numeric score:

```javascript
db.students.aggregate([
  {
    $project: {
      name: 1,
      score: 1,
      grade: {
        $switch: {
          branches: [
            { case: { $gte: ["$score", 90] }, then: "A" },
            { case: { $gte: ["$score", 80] }, then: "B" },
            { case: { $gte: ["$score", 70] }, then: "C" },
            { case: { $gte: ["$score", 60] }, then: "D" }
          ],
          default: "F"
        }
      }
    }
  }
]);
```

Branches are evaluated top-to-bottom. A score of 85 matches `$gte: 80` and returns "B" without evaluating further branches.

## $switch in $group for Multi-Category Counting

Count documents across multiple categories in a single pass:

```javascript
db.transactions.aggregate([
  {
    $group: {
      _id: null,
      microCount:  { $sum: { $cond: [{ $lt:  ["$amount", 10] },   1, 0] } },
      smallCount:  { $sum: { $cond: [{ $lt:  ["$amount", 100] },  1, 0] } },
      mediumCount: { $sum: { $cond: [{ $lt:  ["$amount", 1000] }, 1, 0] } },
      largeCount:  { $sum: { $cond: [{ $gte: ["$amount", 1000] }, 1, 0] } }
    }
  }
]);
```

Or use `$switch` inside `$addFields` to label first, then group:

```javascript
db.transactions.aggregate([
  {
    $addFields: {
      tier: {
        $switch: {
          branches: [
            { case: { $lt:  ["$amount", 10]   }, then: "micro"  },
            { case: { $lt:  ["$amount", 100]  }, then: "small"  },
            { case: { $lt:  ["$amount", 1000] }, then: "medium" }
          ],
          default: "large"
        }
      }
    }
  },
  {
    $group: {
      _id: "$tier",
      count: { $sum: 1 },
      total: { $sum: "$amount" }
    }
  }
]);
```

## Complex Case Expressions

`case` accepts any expression - not just comparisons:

```javascript
db.users.aggregate([
  {
    $project: {
      accessLevel: {
        $switch: {
          branches: [
            {
              case: { $in: ["$role", ["superadmin", "admin"]] },
              then: "full"
            },
            {
              case: {
                $and: [
                  { $eq: ["$role", "editor"] },
                  { $gt: ["$accountAgeDays", 30] }
                ]
              },
              then: "elevated"
            },
            {
              case: { $eq: ["$role", "editor"] },
              then: "standard-editor"
            }
          ],
          default: "read-only"
        }
      }
    }
  }
]);
```

## Handling Missing default

If no branch matches and `default` is omitted, MongoDB throws an error. Always provide a `default` unless you are certain at least one branch always matches:

```javascript
{
  $switch: {
    branches: [...],
    default: "unknown"
  }
}
```

## $switch vs. Nested $cond

For two branches, `$cond` is simpler. For three or more branches, `$switch` is more readable:

```javascript
// Prefer $switch over deeply nested $cond for many branches
```

Both are functionally equivalent - use whichever is cleaner for the number of cases.

## Summary

Use `$switch` when conditional logic requires three or more branches. Branches are evaluated in order; the first matching `case` wins. Always provide a `default` to avoid runtime errors when no case matches. Use `$switch` in `$addFields` to label documents before grouping for cleaner multi-category aggregations.
