# How to Use $function for Custom JavaScript in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $function, Custom JavaScript, Pipeline

Description: Learn how to use $function in MongoDB aggregation to execute custom JavaScript logic per document when built-in operators are insufficient.

---

## Overview

`$function` is a MongoDB aggregation operator (available in 4.4+) that allows you to execute custom JavaScript code as part of an aggregation expression. Unlike `$accumulator` which is for group-level accumulation, `$function` operates on individual documents and can be used in any expression context within a pipeline stage.

## Basic $function Syntax

```javascript
{
  $function: {
    body: <function or string>,
    args: [<expression>, ...],
    lang: "js"
  }
}
```

## Simple Example - Custom String Transformation

```javascript
db.users.aggregate([
  {
    $project: {
      name: 1,
      initials: {
        $function: {
          body: function(name) {
            return name.split(" ")
              .map(function(word) { return word[0].toUpperCase(); })
              .join(".");
          },
          args: ["$name"],
          lang: "js"
        }
      }
    }
  }
])
// "John Doe" -> "J.D"
```

## Custom Date Formatting

MongoDB's built-in date format operators may not cover all formatting needs:

```javascript
db.events.aggregate([
  {
    $project: {
      event: 1,
      formattedDate: {
        $function: {
          body: function(date) {
            const d = new Date(date);
            const days = ["Sun","Mon","Tue","Wed","Thu","Fri","Sat"];
            const months = ["Jan","Feb","Mar","Apr","May","Jun",
                           "Jul","Aug","Sep","Oct","Nov","Dec"];
            return days[d.getDay()] + ", " +
                   months[d.getMonth()] + " " +
                   d.getDate() + " " +
                   d.getFullYear();
          },
          args: ["$eventDate"],
          lang: "js"
        }
      }
    }
  }
])
// Result: "Mon, Jan 15 2024"
```

## Complex Calculation with Multiple Arguments

```javascript
db.loans.aggregate([
  {
    $project: {
      loanId: 1,
      monthlyPayment: {
        $function: {
          body: function(principal, annualRate, termMonths) {
            const r = annualRate / 12 / 100;
            if (r === 0) return principal / termMonths;
            return principal * (r * Math.pow(1 + r, termMonths)) /
                   (Math.pow(1 + r, termMonths) - 1);
          },
          args: ["$principal", "$annualInterestRate", "$termMonths"],
          lang: "js"
        }
      }
    }
  }
])
```

## Using $function for Custom Validation Logic

```javascript
db.orders.aggregate([
  {
    $addFields: {
      validationResult: {
        $function: {
          body: function(items, totalAmount) {
            const computedTotal = items.reduce(function(sum, item) {
              return sum + (item.qty * item.price);
            }, 0);
            const diff = Math.abs(computedTotal - totalAmount);
            return {
              isValid: diff < 0.01,
              computedTotal: computedTotal,
              discrepancy: diff
            };
          },
          args: ["$items", "$totalAmount"],
          lang: "js"
        }
      }
    }
  },
  { $match: { "validationResult.isValid": false } }
])
```

## Custom Slug Generation

```javascript
db.articles.aggregate([
  {
    $project: {
      title: 1,
      slug: {
        $function: {
          body: function(title) {
            return title
              .toLowerCase()
              .replace(/[^a-z0-9\s-]/g, "")
              .replace(/\s+/g, "-")
              .replace(/-+/g, "-")
              .replace(/^-|-$/g, "")
              .substring(0, 80);
          },
          args: ["$title"],
          lang: "js"
        }
      }
    }
  }
])
```

## Passing a Function as a String

The body can be a string representation of a function:

```javascript
db.data.aggregate([
  {
    $project: {
      result: {
        $function: {
          body: "function(x, y) { return Math.sqrt(x*x + y*y); }",
          args: ["$pointX", "$pointY"],
          lang: "js"
        }
      }
    }
  }
])
```

## $function vs $accumulator

```text
| Feature           | $function                    | $accumulator              |
|-------------------|------------------------------|---------------------------|
| Scope             | Per document (expression)    | Per group (accumulator)   |
| Use in $project   | Yes                          | No                        |
| Use in $group     | Yes (as expression)          | Yes (as accumulator)      |
| State between docs| No                           | Yes                       |
| Performance       | Slower than native           | Slower than native        |
```

## Performance Considerations

```text
- $function uses JavaScript execution and is significantly slower than native operators
- Use built-in aggregation operators whenever possible
- $function is best for: complex string manipulation, custom math, validation
- Avoid using $function inside $match - prefer native query operators for filtering
- Test performance with explain() on large collections before deploying
```

## Summary

`$function` lets you run custom JavaScript code as an aggregation expression operating on individual documents, making it useful for complex transformations, custom date formatting, business-specific calculations, and validation logic that built-in operators cannot express. It works in any expression context unlike `$accumulator` which is group-specific. Because of the JavaScript execution overhead, it should be reserved for cases where native MongoDB operators are insufficient.
