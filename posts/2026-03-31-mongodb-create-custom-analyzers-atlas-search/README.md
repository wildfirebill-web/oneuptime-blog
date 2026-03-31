# How to Create Custom Analyzers for Atlas Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, Analyzer, Tokenizer, Full-Text Search

Description: Learn how to build custom analyzers in Atlas Search by combining char filters, tokenizers, and token filters to handle specialized text processing requirements.

---

## When Built-in Analyzers Are Not Enough

Built-in Lucene analyzers handle common cases, but production search often requires specialized behavior: stripping HTML tags, preserving product codes with hyphens, handling domain-specific stop words, or normalizing abbreviations. Custom analyzers let you compose a pipeline from individual components.

## Custom Analyzer Structure

A custom analyzer has three parts:

```json
{
  "charFilters": [],   // optional: pre-tokenization text transformations
  "tokenizer": {},     // required: splits text into tokens
  "tokenFilters": []   // optional: transforms individual tokens
}
```

## Defining Custom Analyzers in an Index

Custom analyzers are defined at the index level and referenced by field mappings:

```json
{
  "analyzers": [
    {
      "name": "productCodeAnalyzer",
      "charFilters": [],
      "tokenizer": {
        "type": "whitespace"
      },
      "tokenFilters": [
        { "type": "lowercase" },
        { "type": "regex", "pattern": "[^a-z0-9-]", "replacement": "", "matches": "all" }
      ]
    },
    {
      "name": "htmlStripAnalyzer",
      "charFilters": [
        { "type": "htmlStrip" }
      ],
      "tokenizer": {
        "type": "standard"
      },
      "tokenFilters": [
        { "type": "lowercase" },
        { "type": "stopword", "tokens": ["the", "a", "an", "and", "or"] }
      ]
    }
  ],
  "mappings": {
    "dynamic": false,
    "fields": {
      "sku": {
        "type": "string",
        "analyzer": "productCodeAnalyzer"
      },
      "htmlDescription": {
        "type": "string",
        "analyzer": "htmlStripAnalyzer"
      }
    }
  }
}
```

## Common Custom Analyzer Recipes

### Autocomplete Analyzer

Uses edge n-grams to enable prefix matching:

```json
{
  "name": "autocompleteAnalyzer",
  "tokenizer": { "type": "standard" },
  "tokenFilters": [
    { "type": "lowercase" },
    { "type": "edgeGram", "minGram": 2, "maxGram": 15 }
  ]
}
```

Index with `autocompleteAnalyzer`, search with `lucene.standard` to avoid n-gram expansion on the query side:

```json
"title": {
  "type": "string",
  "analyzer": "autocompleteAnalyzer",
  "searchAnalyzer": "lucene.standard"
}
```

### Synonym-Aware Analyzer

```json
{
  "name": "synonymAnalyzer",
  "tokenizer": { "type": "standard" },
  "tokenFilters": [
    { "type": "lowercase" },
    { "type": "englishPossessive" },
    { "type": "porterStemming" }
  ]
}
```

Pair this with Atlas Search's synonym mappings for query-time synonym expansion.

## Deploying via Atlas CLI

```bash
atlas clusters search indexes create \
  --clusterName MyCluster \
  --file custom-analyzer-index.json
```

## Testing the Analyzer

Query Atlas Search with a known input to verify tokenization is working as expected:

```javascript
db.products.aggregate([
  { $search: {
    index: "product-search",
    text: {
      query: "SKU-ABC-123",
      path: "sku"
    }
  }},
  { $limit: 5 },
  { $project: { sku: 1, name: 1 } }
]);
```

## Summary

Custom analyzers in Atlas Search are composed of optional char filters, a required tokenizer, and optional token filters. Define them in the `"analyzers"` array at the index level and reference them by name in field mappings. Common patterns include HTML stripping for user-generated content, edge n-gram tokenization for autocomplete, and whitespace-only tokenization for product codes that contain meaningful punctuation.
