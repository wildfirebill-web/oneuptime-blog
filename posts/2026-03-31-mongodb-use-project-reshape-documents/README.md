# How to Use $project to Reshape Documents in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Projection

Description: Learn how to use the $project stage in MongoDB aggregation to include, exclude, rename, and compute fields, reshaping documents for downstream stages or final output.

---

## What Is the $project Stage?

The `$project` stage reshapes documents in an aggregation pipeline. It can include or exclude fields, rename fields, compute new fields with expressions, and transform nested documents. Unlike the `find()` projection, `$project` in aggregation can create entirely new computed fields.

## Basic Syntax

```javascript
db.collection.aggregate([
  {
    $project: {
      <field>: 1,           // include
      <field>: 0,           // exclude
      <newField>: <expr>    // compute
    }
  }
])
```

## Example: Including Specific Fields

```javascript
db.users.aggregate([
  { $project: { name: 1, email: 1, _id: 0 } }
])
// Returns only name and email, no _id
```

## Renaming Fields

```javascript
db.orders.aggregate([
  {
    $project: {
      orderId: "$_id",
      customerName: "$customer.name",
      _id: 0
    }
  }
])
```

## Computing New Fields

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      price: 1,
      discountedPrice: { $multiply: ["$price", 0.9] }
    }
  }
])
```

## Projecting Nested Documents

```javascript
db.employees.aggregate([
  {
    $project: {
      fullName: { $concat: ["$firstName", " ", "$lastName"] },
      "address.city": 1,
      "address.country": 1
    }
  }
])
```

## Conditional Field Computation

```javascript
db.orders.aggregate([
  {
    $project: {
      status: 1,
      priority: {
        $cond: {
          if: { $gte: ["$amount", 1000] },
          then: "high",
          else: "normal"
        }
      }
    }
  }
])
```

## Working with Arrays

```javascript
db.students.aggregate([
  {
    $project: {
      name: 1,
      topScore: { $max: "$scores" },
      scoreCount: { $size: "$scores" }
    }
  }
])
```

## Projecting Array Elements with $arrayElemAt

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      firstImage: { $arrayElemAt: ["$images", 0] },
      lastImage: { $arrayElemAt: ["$images", -1] }
    }
  }
])
```

## Combining $project with $match for Efficiency

```javascript
db.transactions.aggregate([
  { $match: { status: "completed" } },
  {
    $project: {
      userId: 1,
      amount: 1,
      fee: { $multiply: ["$amount", 0.029] },
      net: { $multiply: ["$amount", 0.971] }
    }
  }
])
```

## $project vs $addFields

- `$project` requires you to explicitly include all fields you want to keep (or exclude the ones you don't)
- `$addFields` (also called `$set`) adds new fields while preserving all existing fields

Use `$project` when you want full control over the output shape. Use `$addFields` when you just want to augment documents without listing every field.

## Summary

The `$project` stage is the primary tool for shaping documents in MongoDB aggregation pipelines. It includes and excludes fields, renames them, and computes new values using the full suite of aggregation expressions. Placing `$project` after `$match` reduces document size early, improving performance for downstream stages.
