# How to Use the queryString Operator in MongoDB Atlas Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Search, Query String, Full-Text Search, Atlas

Description: Learn how to use the Atlas Search queryString operator to build search queries with boolean operators and field-specific syntax from user-supplied query strings.

---

The `queryString` operator in MongoDB Atlas Search accepts a Lucene-style query string with boolean operators, field specifiers, wildcards, and phrases. It is designed for search bars where users can type advanced queries like `title:mongodb AND published:2026`.

## Basic queryString Syntax

```javascript
db.articles.aggregate([
  {
    $search: {
      queryString: {
        defaultPath: "body",
        query: "mongodb performance"
      }
    }
  },
  {
    $project: {
      title: 1,
      score: { $meta: "searchScore" }
    }
  }
])
```

Terms separated by spaces are treated as OR by default.

## Boolean Operators

Use AND, OR, NOT to build complex queries:

```javascript
db.articles.aggregate([
  {
    $search: {
      queryString: {
        defaultPath: "body",
        query: "mongodb AND (performance OR optimization) NOT deprecated"
      }
    }
  },
  {
    $project: { title: 1 }
  }
])
```

Supported boolean operators:
- `AND` - both terms must match
- `OR` - either term must match
- `NOT` - exclude documents with the term
- `+term` - term must appear (equivalent to `must`)
- `-term` - term must not appear (equivalent to `mustNot`)

## Field-Specific Search

Target specific fields using `field:value` syntax:

```javascript
db.articles.aggregate([
  {
    $search: {
      queryString: {
        defaultPath: "body",
        query: "title:mongodb author:nawazdhandala"
      }
    }
  },
  {
    $project: { title: 1, author: 1 }
  }
])
```

## Phrase Search

Enclose phrases in double quotes:

```javascript
db.articles.aggregate([
  {
    $search: {
      queryString: {
        defaultPath: "body",
        query: "\"replica set\" AND \"high availability\""
      }
    }
  },
  {
    $project: { title: 1 }
  }
])
```

## Wildcard in queryString

Use `*` and `?` wildcards within the query string:

```javascript
db.products.aggregate([
  {
    $search: {
      queryString: {
        defaultPath: "name",
        query: "wire* head?"
      }
    }
  }
])
```

## Parsing User Input Safely

The `queryString` operator is ideal for exposing advanced search to power users, but invalid syntax returns an error. Always validate or sanitize user input before passing it to the operator:

```javascript
function buildSafeQuery(userInput) {
  // Remove problematic characters
  return userInput.replace(/[<>{}[\]^~]/g, "");
}

const userQuery = buildSafeQuery(req.query.q);

db.articles.aggregate([
  {
    $search: {
      queryString: {
        defaultPath: "body",
        query: userQuery
      }
    }
  },
  { $limit: 20 }
]);
```

## Combining queryString with compound Filters

Pair user query input with server-side filters using `compound`:

```javascript
db.products.aggregate([
  {
    $search: {
      compound: {
        must: [
          {
            queryString: {
              defaultPath: "description",
              query: userQuery
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
  { $limit: 20 }
])
```

## Summary

The `queryString` operator lets users write Lucene-style boolean queries with field specifiers, wildcards, and phrases. It is best suited for advanced search interfaces where users expect Google-like query syntax, and should be combined with `compound` filters to enforce server-side constraints like availability or access control.

