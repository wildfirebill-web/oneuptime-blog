# How to Use the embeddedDocument Operator in MongoDB Atlas Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, Embedded Document, Nested, Atlas

Description: Learn how to use the Atlas Search embeddedDocument operator to search inside arrays of subdocuments while keeping field correlations intact.

---

The `embeddedDocument` operator in MongoDB Atlas Search enables precise searching inside arrays of nested documents. Without it, searching inside an array of subdocuments loses field correlation - a document can match even if the matching fields come from different array elements. The `embeddedDocument` operator solves this by treating each element as an isolated search unit.

## Why embeddedDocument Matters

Consider a product with multiple variants:

```json
{
  "name": "Laptop",
  "variants": [
    { "color": "silver", "size": "15-inch", "inStock": true },
    { "color": "black",  "size": "13-inch", "inStock": false }
  ]
}
```

Without `embeddedDocument`, searching for `color: "silver" AND inStock: false` could incorrectly match this document (silver from element 0, inStock false from element 1). The `embeddedDocument` operator ensures both conditions must be satisfied within the same array element.

## Index Configuration

Enable embedded document indexing in the Atlas Search mapping:

```json
{
  "mappings": {
    "fields": {
      "variants": {
        "type": "embeddedDocuments",
        "fields": {
          "color": {
            "type": "string",
            "analyzer": "lucene.keyword"
          },
          "size": {
            "type": "string",
            "analyzer": "lucene.keyword"
          },
          "inStock": {
            "type": "boolean"
          },
          "price": {
            "type": "number"
          }
        }
      }
    }
  }
}
```

## Basic embeddedDocument Query

Find products that have a silver, in-stock variant:

```javascript
db.products.aggregate([
  {
    $search: {
      embeddedDocument: {
        path: "variants",
        operator: {
          compound: {
            must: [
              {
                equals: {
                  path: "variants.color",
                  value: "silver"
                }
              },
              {
                equals: {
                  path: "variants.inStock",
                  value: true
                }
              }
            ]
          }
        }
      }
    }
  },
  {
    $project: {
      name: 1,
      variants: 1
    }
  }
])
```

## Filtering by Price Within Variants

Find products with a 15-inch variant priced under $1000:

```javascript
db.products.aggregate([
  {
    $search: {
      embeddedDocument: {
        path: "variants",
        operator: {
          compound: {
            must: [
              {
                equals: {
                  path: "variants.size",
                  value: "15-inch"
                }
              }
            ],
            filter: [
              {
                range: {
                  path: "variants.price",
                  lte: 1000
                }
              }
            ]
          }
        }
      }
    }
  },
  {
    $project: {
      name: 1,
      variants: 1,
      score: { $meta: "searchScore" }
    }
  }
])
```

## Combining embeddedDocument with Outer compound

Mix embedded document filtering with outer-level text search:

```javascript
db.products.aggregate([
  {
    $search: {
      compound: {
        must: [
          {
            text: {
              query: "laptop computer",
              path: ["name", "description"]
            }
          }
        ],
        filter: [
          {
            embeddedDocument: {
              path: "variants",
              operator: {
                compound: {
                  must: [
                    { equals: { path: "variants.color", value: "silver" } },
                    { equals: { path: "variants.inStock", value: true } }
                  ]
                }
              }
            }
          }
        ]
      }
    }
  },
  {
    $project: {
      name: 1,
      score: { $meta: "searchScore" }
    }
  }
])
```

## Searching Text Within Subdocuments

Find orders with a line item containing "keyboard":

```javascript
db.orders.aggregate([
  {
    $search: {
      embeddedDocument: {
        path: "lineItems",
        operator: {
          text: {
            query: "mechanical keyboard",
            path: "lineItems.productName"
          }
        }
      }
    }
  },
  {
    $project: {
      orderId: 1,
      lineItems: 1
    }
  }
])
```

## Summary

The `embeddedDocument` operator is essential when searching inside arrays of subdocuments where field correlation matters. By isolating each array element during search evaluation, it prevents false positives from cross-element field matching. Use it with `embeddedDocuments` index mappings and combine it with `compound` for both inner and outer search conditions.

