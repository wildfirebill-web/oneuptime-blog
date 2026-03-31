# How to Build a Multi-Language Content Platform with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Localization, Content, Schema, Index, Internationalization

Description: Learn how to build a multi-language content platform with MongoDB using locale-keyed document fields, collation indexes, and language-specific text search.

---

## Introduction

Building a platform that serves content in multiple languages requires careful schema design. MongoDB supports two main patterns: embedding all translations in a single document, or storing one document per locale. Each approach has trade-offs. This guide covers both patterns and shows how to query, index, and search multilingual content.

## Pattern 1: Embedded Translations

Store all locales in a single document using a `translations` map:

```javascript
{
  _id: ObjectId(),
  slug: "getting-started",
  status: "published",
  defaultLocale: "en",
  translations: {
    en: {
      title: "Getting Started",
      body: "<p>Welcome...</p>",
      excerpt: "Quick start guide"
    },
    fr: {
      title: "Prise en main",
      body: "<p>Bienvenue...</p>",
      excerpt: "Guide de demarrage rapide"
    },
    de: {
      title: "Erste Schritte",
      body: "<p>Willkommen...</p>",
      excerpt: "Schnellstartanleitung"
    }
  },
  publishedAt: ISODate("2026-03-01T00:00:00Z")
}
```

Query a specific locale:

```javascript
const locale = "fr";
db.content.find(
  { status: "published", [`translations.${locale}`]: { $exists: true } },
  { slug: 1, [`translations.${locale}.title`]: 1, publishedAt: 1 }
).sort({ publishedAt: -1 });
```

## Pattern 2: One Document Per Locale

Store each locale as a separate document for simpler queries and indexing:

```javascript
{
  slug: "getting-started",
  locale: "en",
  title: "Getting Started",
  body: "<p>Welcome...</p>",
  status: "published",
  publishedAt: ISODate("2026-03-01T00:00:00Z")
}
```

Indexes for this pattern:

```javascript
db.content.createIndex({ slug: 1, locale: 1 }, { unique: true });
db.content.createIndex({ locale: 1, status: 1, publishedAt: -1 });
```

## Locale-Aware Text Search

Create language-specific text indexes:

```javascript
// English text index
db.createCollection("contentEn");
db.contentEn.createIndex({ title: "text", body: "text" }, { default_language: "english" });

// French text index
db.createCollection("contentFr");
db.contentFr.createIndex({ title: "text", body: "text" }, { default_language: "french" });
```

Search in English:

```javascript
db.contentEn.find({ $text: { $search: "getting started" } });
```

## Collation for Locale-Correct Sorting

Sort French content using French collation:

```javascript
db.content
  .find({ locale: "fr", status: "published" })
  .collation({ locale: "fr", strength: 1 })
  .sort({ title: 1 });
```

## Fallback Strategy

When a translation is unavailable, fall back to the default locale:

```javascript
async function getContent(slug, locale) {
  let doc = await db.collection("content").findOne({ slug, locale });
  if (!doc) {
    doc = await db.collection("content").findOne({ slug, locale: "en" });
  }
  return doc;
}
```

## Summary

MongoDB supports multi-language content effectively using either embedded translation maps or per-locale documents. The per-locale pattern simplifies indexing and text search setup. Language-specific text indexes enable accurate full-text search in each language, while collation indexes ensure locale-correct sort orders. A fallback strategy ensures users always see content even when a translation is missing.
