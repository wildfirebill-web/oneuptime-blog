# How to Use the $ Positional Operator for Array Projection in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array, Projection

Description: Learn how to use the $ positional operator in MongoDB to return only the first matching array element in a query result, reducing data transfer overhead.

---

When querying documents that contain arrays, MongoDB often returns the entire array even if you only need the element that matched your query condition. The `$` positional operator solves this by projecting only the first array element that satisfies the query predicate.

## How the $ Positional Operator Works

The `$` operator acts as a placeholder for the first element in an array that matches the query condition. It works together with the query filter - the field path used in the query must match the field path used in the projection with `$`.

Consider a `students` collection where each document holds an array of grades:

```javascript
db.students.insertMany([
  {
    name: "Alice",
    grades: [
      { subject: "math", score: 92 },
      { subject: "english", score: 78 },
      { subject: "science", score: 85 }
    ]
  },
  {
    name: "Bob",
    grades: [
      { subject: "math", score: 65 },
      { subject: "english", score: 88 },
      { subject: "science", score: 70 }
    ]
  }
])
```

To retrieve each student's math grade only, use:

```javascript
db.students.find(
  { "grades.subject": "math" },
  { name: 1, "grades.$": 1 }
)
```

Result:

```javascript
[
  { name: "Alice", grades: [{ subject: "math", score: 92 }] },
  { name: "Bob",   grades: [{ subject: "math", score: 65 }] }
]
```

Without the `$` projection, both documents would include all three grade subdocuments.

## Rules and Restrictions

The `$` operator has a few important constraints:

- Only one `$` positional projection is allowed per query.
- The array field referenced in the projection must appear in the query filter.
- It returns only the **first** matching element, not all matches.

```javascript
// This will error - cannot use $ without matching query condition
db.students.find(
  {},
  { "grades.$": 1 }  // ERROR: no query condition on grades
)
```

## Combining $ with Other Projections

You can include or exclude other fields alongside the `$` projection:

```javascript
db.students.find(
  { "grades.score": { $gte: 90 } },
  { name: 1, "grades.$": 1, _id: 0 }
)
```

This returns the student name and the first grade with a score of 90 or above, without the `_id` field.

## When to Use vs. $elemMatch Projection

Use the `$` operator when the query already filters on the array field and you want the first matched element. Use `$elemMatch` in the projection when you need to specify different match criteria in the projection itself, independent of the query filter.

```javascript
// $ operator - uses same condition as query
db.students.find(
  { "grades.subject": "math" },
  { "grades.$": 1 }
)

// $elemMatch in projection - can use different conditions
db.students.find(
  { name: "Alice" },
  { grades: { $elemMatch: { subject: "math" } } }
)
```

## Practical Use Case - Returning One Product Variant

In an e-commerce catalog, each product might have many variants. If a user filters by size, you only want to return the matching variant:

```javascript
db.products.find(
  { "variants.size": "L" },
  { name: 1, price: 1, "variants.$": 1 }
)
```

This keeps the response payload small and avoids sending irrelevant variant data to the client.

## Summary

The `$` positional operator in MongoDB projections returns only the first array element that matched the query predicate. It is a lightweight alternative to fetching full arrays when only one matching subdocument is needed. Always pair it with a matching array query condition, and use `$elemMatch` projection when your projection criteria differ from your query filter.
