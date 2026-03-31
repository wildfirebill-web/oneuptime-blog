# How to Use $function for Custom JavaScript in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, JavaScript, $function, Database

Description: Learn how to use $function in MongoDB aggregation to execute custom JavaScript expressions within pipeline stages for computations beyond built-in operators.

---

## Deprecation Notice

> **Warning:** Starting in MongoDB 8.0, `$function` (along with `$accumulator` and `$where`) is deprecated. Prefer native aggregation operators. `$function` continues to work in MongoDB 8.0 but may be removed in a future release.

## Overview

The `$function` operator (introduced in MongoDB 4.4) allows you to define and execute custom JavaScript functions within aggregation pipeline expressions. Unlike `$accumulator` which is used in `$group` stages for accumulation across documents, `$function` is used in expression contexts like `$project`, `$addFields`, and `$match` to compute values on a per-document basis.

## Prerequisites

`$function` requires JavaScript execution to be enabled on the server:

```bash
mongod --setParameter javascriptEnabled=1
```

## Basic Syntax

`$function` takes three fields: `body` (the function), `args` (an array of expression arguments passed to the function), and `lang` (currently only `"js"`).

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      slug: {
        $function: {
          body: function(name) {
            return name.toLowerCase().replace(/\s+/g, '-').replace(/[^a-z0-9-]/g, '');
          },
          args: ["$name"],
          lang: "js"
        }
      }
    }
  }
])
```

## Complex String Formatting

Format a phone number according to custom rules:

```javascript
db.contacts.aggregate([
  {
    $addFields: {
      formattedPhone: {
        $function: {
          body: function(phone) {
            const digits = phone.replace(/\D/g, '');
            if (digits.length === 10) {
              return `(${digits.slice(0,3)}) ${digits.slice(3,6)}-${digits.slice(6)}`;
            }
            return phone;
          },
          args: ["$phone"],
          lang: "js"
        }
      }
    }
  }
])
```

## Using Multiple Arguments

Pass multiple fields as arguments:

```javascript
db.employees.aggregate([
  {
    $addFields: {
      performanceTier: {
        $function: {
          body: function(score, tenure, sales) {
            const tenureBonus = Math.min(tenure * 0.02, 0.1);
            const adjusted = score + tenureBonus + (sales > 100000 ? 0.1 : 0);
            if (adjusted >= 0.9) return "platinum";
            if (adjusted >= 0.75) return "gold";
            if (adjusted >= 0.6) return "silver";
            return "standard";
          },
          args: ["$performanceScore", "$yearsEmployed", "$annualSales"],
          lang: "js"
        }
      }
    }
  }
])
```

## Array Processing with $function

Process an array with custom JavaScript logic:

```javascript
db.orders.aggregate([
  {
    $project: {
      orderId: 1,
      itemSummary: {
        $function: {
          body: function(items) {
            const categorized = items.reduce((acc, item) => {
              const cat = item.category || 'uncategorized';
              if (!acc[cat]) acc[cat] = { count: 0, total: 0 };
              acc[cat].count++;
              acc[cat].total += item.price * item.quantity;
              return acc;
            }, {});
            return Object.entries(categorized).map(([cat, data]) => ({
              category: cat,
              count: data.count,
              total: Math.round(data.total * 100) / 100
            }));
          },
          args: ["$items"],
          lang: "js"
        }
      }
    }
  }
])
```

## Using $function in $match with $expr

Use `$function` in `$match` through `$expr` for complex filtering logic:

```javascript
db.events.aggregate([
  {
    $match: {
      $expr: {
        $function: {
          body: function(tags, requiredTags) {
            return requiredTags.every(tag => tags.includes(tag));
          },
          args: ["$tags", ["mongodb", "aggregation"]],
          lang: "js"
        }
      }
    }
  }
])
```

## When to Use $function vs Built-In Operators

```text
Use built-in operators when possible - they are:
- 10x to 100x faster than JavaScript execution
- Able to leverage indexes
- Compatible with sharded clusters without special considerations

Use $function when:
- Custom string formatting (beyond $concat/$regexReplace)
- Complex business logic with multiple conditions
- Array processing with reduce/groupBy patterns
- Prototype-stage pipelines before optimizing
```

## Performance Warning

JavaScript execution in MongoDB is significantly slower than native operators and cannot use indexes. For production workloads on large collections, always profile and consider rewriting with native operators.

```javascript
// Slow - uses $function for simple multiplication
{ $function: { body: function(a, b) { return a * b; }, args: ["$price", "$quantity"], lang: "js" } }

// Fast - use native operator instead
{ $multiply: ["$price", "$quantity"] }
```

## Summary

The `$function` operator gives MongoDB aggregation access to the full power of JavaScript for per-document computations. It excels at complex string manipulation, custom classification logic, and array processing scenarios where built-in operators are insufficient. However, it carries a significant performance cost, so use it judiciously in development and prototype stages, and replace with native operators in production pipelines wherever possible.
