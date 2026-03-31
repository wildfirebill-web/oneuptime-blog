# How to Use $in (Aggregation Expression) to Check Array Membership in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $in, Array, Pipeline, Expressions

Description: Learn how the $in aggregation expression differs from the $in query operator and how to use it to check if a value exists in an array within aggregation pipelines.

---

## Overview

MongoDB has two distinct uses of `$in`. The query operator `{field: {$in: [...]}}` filters documents. The aggregation expression `{$in: [value, array]}` checks whether a value is present in an array and returns a boolean. This guide focuses on the aggregation expression, which enables conditional logic, computed fields, and filtering based on array membership within `$project`, `$addFields`, and `$match` stages.

## Syntax

```javascript
{ $in: [ <expression>, <array-expression> ] }
```

- First argument: the value to search for
- Second argument: the array to search in
- Returns: `true` if the value is found, `false` otherwise

## Sample Data

```javascript
db.orders.insertMany([
  {
    _id: 1,
    customerId: "C001",
    items: ["laptop", "mouse", "keyboard"],
    status: "shipped",
    tags: ["electronics", "office"]
  },
  {
    _id: 2,
    customerId: "C002",
    items: ["monitor", "webcam"],
    status: "pending",
    tags: ["electronics"]
  },
  {
    _id: 3,
    customerId: "C003",
    items: ["desk", "chair"],
    status: "delivered",
    tags: ["furniture", "office"]
  }
]);
```

## Basic Usage in $project

Check if a specific value is in an array field:

```javascript
db.orders.aggregate([
  {
    $project: {
      customerId: 1,
      items: 1,
      hasLaptop: {
        $in: ["laptop", "$items"]
      }
    }
  }
]);
```

```text
{ customerId: "C001", items: [...], hasLaptop: true }
{ customerId: "C002", items: [...], hasLaptop: false }
{ customerId: "C003", items: [...], hasLaptop: false }
```

## Checking Dynamic Values

Check whether one document field appears in an array field from the same document:

```javascript
db.employees.aggregate([
  {
    $project: {
      name: 1,
      department: 1,
      isApprover: {
        $in: ["$department", "$approvalDepartments"]
      }
    }
  }
]);
```

Both arguments can be expressions, enabling cross-field membership checks.

## Using $in in $addFields

Add a computed boolean field without removing other fields:

```javascript
db.orders.aggregate([
  {
    $addFields: {
      isOfficeOrder: {
        $in: ["office", "$tags"]
      },
      isElectronics: {
        $in: ["electronics", "$tags"]
      }
    }
  }
]);
```

```text
{ _id: 1, ..., isOfficeOrder: true, isElectronics: true }
{ _id: 2, ..., isOfficeOrder: false, isElectronics: true }
{ _id: 3, ..., isOfficeOrder: true, isElectronics: false }
```

## Using $in with $cond for Conditional Logic

Apply different logic based on array membership:

```javascript
db.orders.aggregate([
  {
    $project: {
      customerId: 1,
      status: 1,
      priority: {
        $cond: {
          if: { $in: ["$status", ["shipped", "delivered"]] },
          then: "low",
          else: "high"
        }
      }
    }
  }
]);
```

```text
{ customerId: "C001", status: "shipped", priority: "low" }
{ customerId: "C002", status: "pending", priority: "high" }
{ customerId: "C003", status: "delivered", priority: "low" }
```

## Using $in in $match Within an Aggregation Pipeline

The aggregation expression `$in` can be used with `$expr` inside a `$match` stage:

```javascript
db.orders.aggregate([
  {
    $match: {
      $expr: {
        $in: ["laptop", "$items"]
      }
    }
  }
]);
```

This returns only orders that include "laptop" in the items array.

Compare this to the query operator syntax (which does NOT use `$expr`):

```javascript
// Query operator - works at find() level and $match without $expr
db.orders.find({ items: { $in: ["laptop"] } });

// Aggregation expression - works with $expr and inside computed fields
db.orders.aggregate([
  { $match: { $expr: { $in: ["laptop", "$items"] } } }
]);
```

## Combining $in with $filter and $map

Use `$in` inside `$filter` to select matching array elements:

```javascript
db.orders.aggregate([
  {
    $project: {
      customerId: 1,
      electronicsItems: {
        $filter: {
          input: "$items",
          as: "item",
          cond: {
            $in: ["$$item", ["laptop", "mouse", "keyboard", "monitor", "webcam"]]
          }
        }
      }
    }
  }
]);
```

```text
{ customerId: "C001", electronicsItems: ["laptop", "mouse", "keyboard"] }
{ customerId: "C002", electronicsItems: ["monitor", "webcam"] }
{ customerId: "C003", electronicsItems: [] }
```

## $in with a Computed Array

The second argument to `$in` can itself be a computed expression, not just a literal array or field reference:

```javascript
db.products.aggregate([
  {
    $addFields: {
      allowedCategories: { $literal: ["electronics", "furniture", "appliances"] }
    }
  },
  {
    $project: {
      name: 1,
      category: 1,
      isAllowed: {
        $in: ["$category", "$allowedCategories"]
      }
    }
  }
]);
```

## Practical Example: Role-Based Access Check

```javascript
db.users.aggregate([
  {
    $project: {
      username: 1,
      roles: 1,
      canPublish: {
        $in: ["editor", "$roles"]
      },
      canAdmin: {
        $in: ["admin", "$roles"]
      },
      accessLevel: {
        $switch: {
          branches: [
            {
              case: { $in: ["admin", "$roles"] },
              then: "full"
            },
            {
              case: { $in: ["editor", "$roles"] },
              then: "edit"
            },
            {
              case: { $in: ["viewer", "$roles"] },
              then: "read"
            }
          ],
          default: "none"
        }
      }
    }
  }
]);
```

## Summary

The `$in` aggregation expression checks whether a value exists in an array and returns a boolean, making it useful for conditional fields, filtering within pipelines, and array membership logic in `$project`, `$addFields`, and `$match` stages. It differs from the `$in` query operator (which filters documents) and is especially powerful when combined with `$cond`, `$filter`, `$switch`, and `$expr`. Both the search value and the array can be expressions, enabling dynamic membership checks across fields.
