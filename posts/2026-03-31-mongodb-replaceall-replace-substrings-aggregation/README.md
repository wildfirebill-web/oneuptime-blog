# How to Replace Substrings in MongoDB Aggregation with $replaceAll

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, String, Pipeline, Transformation

Description: Learn how to replace substrings in MongoDB aggregation using $replaceAll and $replaceOne, with examples for sanitizing data and building slug-friendly strings.

---

MongoDB provides two aggregation operators for substring replacement: `$replaceOne`, which replaces the first occurrence, and `$replaceAll`, which replaces every occurrence. Both were introduced in MongoDB 4.4 and work within any pipeline stage that accepts expressions.

## $replaceAll Syntax

```javascript
{ $replaceAll: { input: <expression>, find: <string>, replacement: <string> } }
```

All three fields are required. `input` and `find` can be field references or string literals. `replacement` specifies what to substitute.

## Replacing Spaces with Hyphens

A common use case is converting human-readable titles into URL slugs.

```javascript
db.articles.aggregate([
  {
    $project: {
      slug: {
        $replaceAll: {
          input: { $toLower: "$title" },
          find: " ",
          replacement: "-"
        }
      }
    }
  }
]);
```

## $replaceOne for Single Replacement

Use `$replaceOne` when only the first occurrence should be replaced.

```javascript
db.logs.aggregate([
  {
    $project: {
      sanitized: {
        $replaceOne: {
          input: "$message",
          find: "ERROR",
          replacement: "WARN"
        }
      }
    }
  }
]);
```

## Chaining Multiple Replacements

Nest `$replaceAll` calls to handle multiple patterns in sequence.

```javascript
db.articles.aggregate([
  {
    $addFields: {
      slug: {
        $replaceAll: {
          input: {
            $replaceAll: {
              input: { $toLower: "$title" },
              find: " ",
              replacement: "-"
            }
          },
          find: "/",
          replacement: ""
        }
      }
    }
  }
]);
```

The inner call converts spaces to hyphens; the outer call removes forward slashes.

## Handling Null Input

If `input` is null or the field is missing, `$replaceAll` returns null. Guard with `$ifNull`.

```javascript
db.products.aggregate([
  {
    $project: {
      cleanDescription: {
        $replaceAll: {
          input: { $ifNull: ["$description", ""] },
          find: "\n",
          replacement: " "
        }
      }
    }
  }
]);
```

## Using $replaceAll in $set with updateMany

Apply `$replaceAll` as part of an update pipeline to modify existing documents.

```javascript
db.articles.updateMany(
  {},
  [
    {
      $set: {
        title: {
          $replaceAll: {
            input: "$title",
            find: "  ",
            replacement: " "
          }
        }
      }
    }
  ]
);
```

Update pipelines (array syntax) support aggregation expressions including `$replaceAll`, which lets you transform data in place without a separate read-modify-write cycle.

## Summary

`$replaceAll` replaces every occurrence of a substring within a string field in MongoDB aggregation. Use it with `$toLower` to build URL slugs, chain nested calls for multiple replacements, and apply it inside update pipelines to clean data at scale. Use `$replaceOne` when only the first match should be changed. Always guard against null inputs with `$ifNull` to prevent null propagation through the pipeline.
