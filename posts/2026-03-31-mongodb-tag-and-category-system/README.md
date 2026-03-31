# How to Build a Tag and Category System in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Tagging, Array, Index, Aggregation

Description: Learn how to implement a flexible tag and category system in MongoDB using array fields, multikey indexes, and aggregation for taxonomy-based filtering.

---

Tags and categories are fundamental to content organization. MongoDB's array fields and multikey indexes make it straightforward to attach multiple tags to any document and efficiently query by one or many tags.

## Embedding Tags in Documents

Store tags as a simple string array directly on the content document.

```javascript
db.articles.insertOne({
  title: "Getting Started with Kubernetes",
  body: "...",
  category: "infrastructure",
  tags: ["kubernetes", "devops", "containers", "orchestration"],
  author: "alice",
  publishedAt: new Date()
});
```

Categories are represented as a single string field when each document belongs to exactly one category. Use an array if multiple categories are needed.

## Creating a Multikey Index on Tags

MongoDB automatically creates a multikey index when you index an array field. This lets you query for any tag value efficiently.

```javascript
db.articles.createIndex({ tags: 1 });
db.articles.createIndex({ category: 1, publishedAt: -1 });
```

## Querying by a Single Tag

Find all articles with a specific tag.

```javascript
db.articles.find(
  { tags: "kubernetes" },
  { title: 1, category: 1, publishedAt: 1 }
).sort({ publishedAt: -1 });
```

## Querying by Multiple Tags

Use `$all` to require all specified tags to be present, or `$in` to match any of them.

```javascript
// Articles that have BOTH tags
db.articles.find({ tags: { $all: ["kubernetes", "devops"] } });

// Articles that have ANY of these tags
db.articles.find({ tags: { $in: ["kubernetes", "docker"] } });
```

## Counting Articles per Tag

Use `$unwind` followed by `$group` to build a tag frequency report.

```javascript
db.articles.aggregate([
  { $unwind: "$tags" },
  {
    $group: {
      _id: "$tags",
      count: { $sum: 1 }
    }
  },
  { $sort: { count: -1 } },
  { $limit: 20 }
]);
```

`$unwind` deconstructs the tags array so that each tag becomes a separate pipeline document, which `$group` then aggregates into per-tag counts.

## Managing a Separate Tags Collection

For tag metadata such as descriptions or aliases, maintain a dedicated `tags` collection.

```javascript
db.tags.insertMany([
  { slug: "kubernetes", label: "Kubernetes", relatedTags: ["containers", "devops"] },
  { slug: "devops", label: "DevOps", relatedTags: ["ci-cd", "automation"] }
]);
db.tags.createIndex({ slug: 1 }, { unique: true });
```

Look up tag metadata with a `$lookup` stage when you need to return enriched tag objects alongside article results.

## Summary

A tag and category system in MongoDB is built around storing tags as string arrays on content documents, backed by a multikey index for fast per-tag filtering. Use `$all` for AND semantics and `$in` for OR semantics when querying. The `$unwind` plus `$group` pattern produces tag frequency reports suitable for tag clouds or related-content recommendations.
