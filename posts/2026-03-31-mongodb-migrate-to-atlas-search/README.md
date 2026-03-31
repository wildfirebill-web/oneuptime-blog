# How to Migrate from Elasticsearch to MongoDB Atlas Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, Elasticsearch, Migration, Search

Description: Learn how to migrate from a separate Elasticsearch cluster to MongoDB Atlas Search to consolidate your search and database infrastructure and simplify operations.

---

## Overview

Running Elasticsearch alongside MongoDB adds operational complexity. MongoDB Atlas Search is a full-text search engine built on Apache Lucene that runs inside MongoDB Atlas, eliminating the need for a separate search cluster. This guide covers replicating your Elasticsearch functionality with Atlas Search.

## Step 1: Create an Atlas Search Index

In MongoDB Atlas, navigate to your collection and create a search index:

```json
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "name": {
        "type": "string",
        "analyzer": "lucene.standard"
      },
      "description": {
        "type": "string",
        "analyzer": "lucene.standard"
      },
      "category": {
        "type": "string",
        "analyzer": "lucene.keyword"
      },
      "price": {
        "type": "number"
      }
    }
  }
}
```

Or via the Atlas CLI:

```bash
atlas clusters search indexes create \
  --clusterName MyCluster \
  --db mydb \
  --collection products \
  --file search-index.json
```

## Step 2: Replace Elasticsearch Queries with $search

Elasticsearch multi-field search:

```json
{
  "query": {
    "multi_match": {
      "query": "wireless headphones",
      "fields": ["name", "description"]
    }
  }
}
```

Equivalent Atlas Search query:

```javascript
db.products.aggregate([
  { $search: {
      index: "products_search",
      text: {
        query: "wireless headphones",
        path: ["name", "description"]
      }
  }},
  { $limit: 10 },
  { $project: { name: 1, description: 1, price: 1, score: { $meta: "searchScore" } } }
])
```

## Step 3: Replicate Autocomplete

Elasticsearch prefix/completion suggester becomes Atlas Search autocomplete:

```javascript
db.products.aggregate([
  { $search: {
      index: "products_search",
      autocomplete: {
        query: "wire",
        path: "name",
        fuzzy: { maxEdits: 1 }
      }
  }},
  { $limit: 5 },
  { $project: { name: 1 } }
])
```

## Step 4: Replicate Faceted Search

Elasticsearch aggregations for facets:

```javascript
db.products.aggregate([
  { $searchMeta: {
      index: "products_search",
      facet: {
        operator: {
          text: { query: "headphones", path: "name" }
        },
        facets: {
          categoryFacet: {
            type: "string",
            path: "category",
            numBuckets: 10
          },
          priceFacet: {
            type: "number",
            path: "price",
            boundaries: [0, 50, 100, 200, 500]
          }
        }
      }
  }}
])
```

## Step 5: Data Migration

Your data is already in MongoDB. Create the Atlas Search index and it automatically indexes existing documents. No data migration is needed - just replace your application's Elasticsearch query calls with `$search` aggregation stages.

## Step 6: Deprecate the Elasticsearch Cluster

Once you have verified that Atlas Search results match your Elasticsearch results, remove the Elasticsearch sync pipeline (Monstache, Kafka connector, or custom change stream) and decommission the Elasticsearch cluster.

## Summary

Migrating from Elasticsearch to MongoDB Atlas Search eliminates a separate search cluster by moving full-text search directly into MongoDB. Create an Atlas Search index on your existing collection, replace Elasticsearch queries with `$search` aggregation stages, and verify results before decommissioning the Elasticsearch infrastructure. There is no data migration since the data is already in MongoDB.
