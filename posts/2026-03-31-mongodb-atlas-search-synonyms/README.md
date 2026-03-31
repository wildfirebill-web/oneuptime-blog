# How to Use Synonyms in MongoDB Atlas Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, Full-Text Search

Description: Configure synonym mappings in MongoDB Atlas Search so that queries for equivalent terms return consistent results without requiring users to know exact field values.

---

Atlas Search synonyms let you expand a user's search query to include related terms automatically. When a user searches for "sneakers," they also get results matching "trainers" or "athletic shoes" - without you changing any document data.

## Synonym Mapping Types

Atlas Search supports two synonym mapping styles:

- **Equivalent** - all listed terms are interchangeable
- **Explicit** - a set of input terms maps to a set of output terms (one-directional)

## Storing Synonym Documents

Synonym mappings live in a regular MongoDB collection. Each document represents one mapping:

```javascript
// Equivalent synonyms - all terms match each other
db.search_synonyms.insertMany([
  {
    mappingType: "equivalent",
    synonyms: ["sneakers", "trainers", "athletic shoes", "running shoes"]
  },
  {
    mappingType: "equivalent",
    synonyms: ["television", "TV", "telly", "flat screen"]
  }
])

// Explicit synonyms - input terms expand to output terms
db.search_synonyms.insertOne({
  mappingType: "explicit",
  input: ["laptop"],
  synonyms: ["notebook", "portable computer", "laptop computer"]
})
```

## Configuring Synonyms in the Atlas Search Index

Reference the synonym source collection in your index definition. Give the mapping a name you will use at query time:

```json
{
  "mappings": {
    "dynamic": true
  },
  "synonyms": [
    {
      "name": "product_synonyms",
      "analyzer": "lucene.standard",
      "source": {
        "collection": "search_synonyms"
      }
    }
  ]
}
```

The `analyzer` must match the analyzer used on the field being queried.

## Querying with Synonyms

Pass the synonym mapping name in the `synonyms` option of a `text` operator:

```javascript
db.products.aggregate([
  {
    $search: {
      index: "default",
      text: {
        query: "sneakers",
        path: "name",
        synonyms: "product_synonyms"
      }
    }
  },
  { $limit: 10 },
  { $project: { name: 1, score: { $meta: "searchScore" } } }
])
```

This returns documents containing "sneakers", "trainers", "athletic shoes", or "running shoes".

## Updating Synonyms Without Re-indexing

Unlike most Atlas Search index changes, synonym updates take effect automatically - you do not need to rebuild the index. Just update the documents in the synonym collection:

```javascript
db.search_synonyms.updateOne(
  { mappingType: "equivalent", synonyms: { $elemMatch: { $eq: "TV" } } },
  { $addToSet: { synonyms: "smart TV" } }
)
```

Atlas Search picks up the change within a few seconds.

## Multiple Synonym Sets

You can define multiple synonym mappings in one index and use different mappings for different queries:

```json
"synonyms": [
  {
    "name": "product_synonyms",
    "analyzer": "lucene.standard",
    "source": { "collection": "product_synonyms_collection" }
  },
  {
    "name": "brand_synonyms",
    "analyzer": "lucene.standard",
    "source": { "collection": "brand_synonyms_collection" }
  }
]
```

Then at query time, select the appropriate mapping per field.

## Limitations

- Synonyms only work with the `text` operator, not `phrase` or `regex`.
- The synonym analyzer must match the field's indexed analyzer exactly.
- Synonym expansion can affect performance on very large synonym sets - keep the collection focused on relevant terms.

## Summary

Atlas Search synonyms improve search recall by automatically expanding queries to include equivalent terms. Store synonym mappings in a standard MongoDB collection, reference the collection in the index definition, and pass the mapping name in your `text` queries. Updates to the synonym collection take effect without any index rebuild, making synonym management straightforward at runtime.
