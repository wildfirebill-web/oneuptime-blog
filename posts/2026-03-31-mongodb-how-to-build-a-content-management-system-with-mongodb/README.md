# How to Build a Content Management System with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Content Management, Schema, Index, Aggregation, Publishing

Description: Learn how to build a full-featured content management system with MongoDB, covering content types, versioning, publishing workflows, and search.

---

## Introduction

MongoDB's flexible document model is well suited to CMS applications where content types evolve over time and rich metadata must be stored alongside content. This guide covers building a CMS backend with MongoDB, including multi-content-type support, versioning, and publishing workflows.

## Content Schema

```javascript
{
  _id: ObjectId(),
  contentType: "article",
  slug: "intro-to-mongodb-cms",
  title: "Introduction to MongoDB CMS",
  status: "published",
  version: 3,
  locale: "en",
  author: { id: ObjectId(), name: "Alice" },
  body: "<p>Content here...</p>",
  excerpt: "A short summary",
  tags: ["mongodb", "cms", "tutorial"],
  category: "technology",
  seo: { title: "Intro to MongoDB CMS", description: "..." },
  publishedAt: ISODate("2026-03-01T00:00:00Z"),
  createdAt: ISODate("2026-01-01T00:00:00Z"),
  updatedAt: ISODate("2026-03-01T00:00:00Z")
}
```

## Versioning with a History Collection

Store content revisions in a separate collection:

```javascript
// Save current version before update
async function updateContent(slug, updates) {
  const current = await db.content.findOne({ slug });
  await db.contentHistory.insertOne({
    contentId: current._id,
    slug,
    version: current.version,
    body: current.body,
    archivedAt: new Date()
  });

  return db.content.findOneAndUpdate(
    { slug },
    {
      $set: { ...updates, updatedAt: new Date() },
      $inc: { version: 1 }
    },
    { returnDocument: "after" }
  );
}
```

## Publishing Workflow

Move content through draft, review, and published states:

```javascript
function publish(slug) {
  return db.content.updateOne(
    { slug, status: { $in: ["draft", "review"] } },
    {
      $set: {
        status: "published",
        publishedAt: new Date(),
        updatedAt: new Date()
      }
    }
  );
}
```

## Listing Published Content with Pagination

```javascript
function listPublished(locale, page = 1, pageSize = 10) {
  return db.content.find(
    { status: "published", locale },
    { title: 1, slug: 1, excerpt: 1, publishedAt: 1, tags: 1 }
  )
    .sort({ publishedAt: -1 })
    .skip((page - 1) * pageSize)
    .limit(pageSize)
    .toArray();
}
```

## Full-Text Search

```javascript
db.content.createIndex({ title: "text", body: "text", tags: "text" });

db.content.find(
  { $text: { $search: "mongodb tutorial" }, status: "published" },
  { score: { $meta: "textScore" }, title: 1, slug: 1, excerpt: 1 }
).sort({ score: { $meta: "textScore" } });
```

## Content Indexes

```javascript
db.content.createIndex({ slug: 1, locale: 1 }, { unique: true });
db.content.createIndex({ status: 1, locale: 1, publishedAt: -1 });
db.content.createIndex({ tags: 1, status: 1 });
db.content.createIndex({ category: 1, status: 1, publishedAt: -1 });
```

## Summary

MongoDB provides all the building blocks needed for a modern CMS: flexible document schemas for varied content types, atomic updates for safe publishing workflows, versioning via a history collection, and text indexes for full-text search. This architecture supports content teams working on articles, pages, and multimedia without database schema changes as content types evolve.
