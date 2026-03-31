# How to Use the moreLikeThis Operator in MongoDB Atlas Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, MoreLikeThis, Recommendation, Atlas

Description: Learn how to use the Atlas Search moreLikeThis operator to find documents similar to a given document, enabling content-based recommendations in MongoDB.

---

The `moreLikeThis` operator in MongoDB Atlas Search finds documents similar to one or more input documents based on their textual content. It is the foundation for "related articles", "similar products", and "you might also like" features, without requiring a separate ML model.

## How moreLikeThis Works

The operator extracts the most significant terms from the input document(s) and constructs a query to find other documents containing those terms. It uses term frequency and inverse document frequency (TF-IDF) to identify representative terms.

## Basic Usage

Find articles similar to the one with a known `_id`:

```javascript
// Step 1: Fetch the source document
const source = db.articles.findOne({ _id: ObjectId("64a1f2c3b4e5f6789abc1234") });

// Step 2: Find similar articles
db.articles.aggregate([
  {
    $search: {
      moreLikeThis: {
        like: [
          {
            title: source.title,
            body: source.body,
            tags: source.tags
          }
        ]
      }
    }
  },
  {
    $match: {
      _id: { $ne: source._id }
    }
  },
  {
    $project: {
      title: 1,
      score: { $meta: "searchScore" }
    }
  },
  { $limit: 5 }
])
```

## Using Multiple Input Documents

Pass several documents to find content similar to any of them:

```javascript
const docs = db.articles.find({ category: "databases" }).limit(3).toArray();

db.articles.aggregate([
  {
    $search: {
      moreLikeThis: {
        like: docs.map(d => ({ title: d.title, body: d.body }))
      }
    }
  },
  {
    $project: {
      title: 1,
      score: { $meta: "searchScore" }
    }
  },
  { $limit: 10 }
])
```

## Similar Products Recommendation

For an e-commerce recommendation engine:

```javascript
db.products.aggregate([
  {
    $search: {
      compound: {
        must: [
          {
            moreLikeThis: {
              like: [
                {
                  name: "Wireless Noise Cancelling Headphones",
                  description: "Over-ear bluetooth headphones with active noise cancellation",
                  category: "electronics"
                }
              ]
            }
          }
        ],
        filter: [
          {
            equals: {
              path: "inStock",
              value: true
            }
          }
        ]
      }
    }
  },
  {
    $project: {
      name: 1,
      price: 1,
      score: { $meta: "searchScore" }
    }
  },
  { $limit: 6 }
])
```

## Using moreLikeThis with a Raw Text Input

You can pass arbitrary text rather than stored documents:

```javascript
db.articles.aggregate([
  {
    $search: {
      moreLikeThis: {
        like: [
          {
            content: "NoSQL databases offer flexible schemas and horizontal scaling for modern applications"
          }
        ]
      }
    }
  },
  {
    $project: {
      title: 1,
      score: { $meta: "searchScore" }
    }
  },
  { $limit: 5 }
])
```

## Controlling Field Weights

Combine `moreLikeThis` with `compound` to boost matches in certain fields:

```javascript
db.articles.aggregate([
  {
    $search: {
      compound: {
        should: [
          {
            moreLikeThis: {
              like: [{ title: sourceDoc.title, body: sourceDoc.body }]
            }
          },
          {
            moreLikeThis: {
              like: [{ tags: sourceDoc.tags }],
              score: { boost: { value: 2 } }
            }
          }
        ]
      }
    }
  },
  { $limit: 5 }
])
```

## Index Requirements

Fields used in `moreLikeThis` must be indexed as `string` with a text analyzer:

```json
{
  "mappings": {
    "fields": {
      "title": { "type": "string", "analyzer": "lucene.standard" },
      "body": { "type": "string", "analyzer": "lucene.english" }
    }
  }
}
```

## Summary

The `moreLikeThis` operator enables similarity-based search in Atlas by extracting significant terms from input documents and matching them across the collection. It is the simplest way to add content-based recommendations to MongoDB-backed applications without an external ML pipeline.

