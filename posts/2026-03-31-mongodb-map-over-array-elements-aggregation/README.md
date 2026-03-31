# How to Map Over Array Elements in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array, Aggregation

Description: Learn how to transform every element of an array in MongoDB using the $map aggregation operator with practical transformation examples.

---

The `$map` operator in MongoDB aggregation applies an expression to each element of an array and returns a new array of transformed values. It is the MongoDB equivalent of `Array.map()` in JavaScript or `list comprehension` in Python.

## Basic $map Syntax

```javascript
{
  $map: {
    input: <array expression>,
    as: <variable name>,
    in: <expression using $$variable>
  }
}
```

- `input` is the array to iterate over
- `as` names the variable holding the current element (referenced with `$$`)
- `in` is the expression applied to each element

## Transforming Scalar Values

Multiply every element in a `scores` array by 1.1 (a 10% bonus):

```javascript
db.students.aggregate([
  {
    $project: {
      name: 1,
      adjustedScores: {
        $map: {
          input: "$scores",
          as: "score",
          in: { $multiply: ["$$score", 1.1] }
        }
      }
    }
  }
]);
```

## Extracting a Field from an Array of Subdocuments

A very common use case is extracting one field from each subdocument to produce a flat array:

```javascript
// Extract just the productId from each line item
db.orders.aggregate([
  {
    $project: {
      productIds: {
        $map: {
          input: "$lineItems",
          as: "item",
          in: "$$item.productId"
        }
      }
    }
  }
]);
```

Result: `{ productIds: ["P-001", "P-002", "P-003"] }`

## Transforming Subdocuments

Apply a transformation to each embedded document - for example, adding a computed field to each element:

```javascript
db.orders.aggregate([
  {
    $project: {
      lineItems: {
        $map: {
          input: "$lineItems",
          as: "item",
          in: {
            productId: "$$item.productId",
            qty: "$$item.qty",
            price: "$$item.price",
            subtotal: { $multiply: ["$$item.qty", "$$item.price"] }
          }
        }
      }
    }
  }
]);
```

Each element in the output array is an enriched version of the input element.

## Applying Conditional Logic in $map

Combine `$map` with `$cond` to apply different transformations based on element value:

```javascript
db.products.aggregate([
  {
    $project: {
      normalizedTags: {
        $map: {
          input: "$tags",
          as: "tag",
          in: { $toLower: "$$tag" }
        }
      }
    }
  }
]);
```

Or a conditional:

```javascript
// Replace "sale" with "on-sale", keep others as-is
db.products.aggregate([
  {
    $project: {
      updatedTags: {
        $map: {
          input: "$tags",
          as: "tag",
          in: {
            $cond: {
              if: { $eq: ["$$tag", "sale"] },
              then: "on-sale",
              else: "$$tag"
            }
          }
        }
      }
    }
  }
]);
```

## Chaining $map and $filter

`$map` and `$filter` compose naturally - transform then filter, or filter then transform:

```javascript
// Keep only items over $50, then compute their discounted price
db.orders.aggregate([
  {
    $project: {
      discountedItems: {
        $map: {
          input: {
            $filter: {
              input: "$lineItems",
              as: "item",
              cond: { $gt: ["$$item.price", 50] }
            }
          },
          as: "item",
          in: {
            productId: "$$item.productId",
            discountedPrice: { $multiply: ["$$item.price", 0.9] }
          }
        }
      }
    }
  }
]);
```

## Summary

Use `$map` to transform every element of an array within an aggregation pipeline. It applies any aggregation expression - arithmetic, string manipulation, conditional logic, subdocument construction - to each element. Combine `$map` with `$filter`, `$reduce`, or other array operators to build sophisticated array processing pipelines.
