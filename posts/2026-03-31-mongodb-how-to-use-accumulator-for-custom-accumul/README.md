# How to Use $accumulator for Custom Accumulation Logic in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Custom Accumulator, JavaScript, Database

Description: Learn how to use $accumulator in MongoDB aggregation to define custom JavaScript accumulation logic for group stages beyond the built-in accumulators.

---

## Overview

MongoDB's `$accumulator` operator (introduced in MongoDB 4.4) allows you to define custom accumulation logic using JavaScript for the `$group` stage. When built-in accumulators like `$sum`, `$avg`, `$min`, and `$max` are insufficient for your use case, `$accumulator` lets you write an `init`, `accumulate`, `merge`, and optional `finalize` function to implement any accumulation behavior.

## Prerequisites

`$accumulator` requires server-side JavaScript to be enabled. Enable it in `mongod.conf`:

```yaml
security:
  javascriptEnabled: true
```

Or as a startup flag:

```bash
mongod --setParameter javascriptEnabled=1
```

## Basic Syntax

`$accumulator` takes four main functions:

- `init`: initializes the accumulator state
- `accumulate`: processes each document and updates the state
- `accumulateArgs`: fields to pass to the accumulate function
- `merge`: merges two accumulator states (for sharded environments)
- `finalize` (optional): transforms the final state into the output value
- `lang`: the scripting language (currently only `"js"`)

## Example: Custom Variance Calculation

Calculate variance across a group using Welford's online algorithm:

```javascript
db.measurements.aggregate([
  {
    $group: {
      _id: "$sensorId",
      variance: {
        $accumulator: {
          init: function() {
            return { count: 0, mean: 0, M2: 0 };
          },
          accumulate: function(state, value) {
            state.count++;
            const delta = value - state.mean;
            state.mean += delta / state.count;
            const delta2 = value - state.mean;
            state.M2 += delta * delta2;
            return state;
          },
          accumulateArgs: ["$reading"],
          merge: function(state1, state2) {
            const count = state1.count + state2.count;
            const delta = state2.mean - state1.mean;
            const mean = (state1.count * state1.mean + state2.count * state2.mean) / count;
            const M2 = state1.M2 + state2.M2 + (delta * delta * state1.count * state2.count) / count;
            return { count, mean, M2 };
          },
          finalize: function(state) {
            return state.count < 2 ? 0 : state.M2 / (state.count - 1);
          },
          lang: "js"
        }
      }
    }
  }
])
```

## Example: Collecting Unique Values

Collect a set of unique values (like a distinct per group):

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$customerId",
      uniqueProducts: {
        $accumulator: {
          init: function() { return []; },
          accumulate: function(state, productId) {
            if (!state.includes(productId)) {
              state.push(productId);
            }
            return state;
          },
          accumulateArgs: ["$productId"],
          merge: function(state1, state2) {
            return state1.concat(state2.filter(v => !state1.includes(v)));
          },
          lang: "js"
        }
      }
    }
  }
])
```

## Example: Most Frequent Value (Mode)

Find the mode of a field within each group:

```javascript
db.survey.aggregate([
  {
    $group: {
      _id: "$questionId",
      mostFrequent: {
        $accumulator: {
          init: function() { return {}; },
          accumulate: function(state, answer) {
            state[answer] = (state[answer] || 0) + 1;
            return state;
          },
          accumulateArgs: ["$answer"],
          merge: function(s1, s2) {
            for (const key of Object.keys(s2)) {
              s1[key] = (s1[key] || 0) + s2[key];
            }
            return s1;
          },
          finalize: function(state) {
            return Object.entries(state).sort((a, b) => b[1] - a[1])[0][0];
          },
          lang: "js"
        }
      }
    }
  }
])
```

## Performance Considerations

`$accumulator` runs JavaScript in the mongod process which is significantly slower than native operators. Consider these points:

```text
- Use built-in operators whenever possible ($sum, $avg, $push, $addToSet)
- $accumulator is best for logic that truly cannot be expressed with built-in operators
- Avoid in high-throughput or large-dataset pipelines
- Test performance on representative data volumes before production deployment
```

## Summary

The `$accumulator` operator is a powerful escape hatch for custom aggregation logic that goes beyond MongoDB's built-in accumulators. By defining `init`, `accumulate`, `merge`, and `finalize` functions in JavaScript, you can implement statistical functions, unique value sets, mode calculations, and any stateful accumulation. Reserve it for cases where built-in operators fall short, as it carries a performance cost compared to native MongoDB expressions.
