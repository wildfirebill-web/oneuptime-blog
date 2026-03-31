# How to Use $covariancePop and $covarianceSamp in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Window Function, Covariance, Analytics

Description: Learn how to compute population and sample covariance in MongoDB using $covariancePop and $covarianceSamp operators inside $setWindowFields for statistical analysis.

---

MongoDB's `$covariancePop` and `$covarianceSamp` operators compute covariance between two numeric fields across a window of documents. Covariance measures how two variables change together - positive covariance means they increase together, negative means one increases as the other decreases.

## Population vs Sample Covariance

- `$covariancePop` - population covariance, divides by N (use when the window represents the entire population)
- `$covarianceSamp` - sample covariance, divides by N-1 (use when the window is a sample; preferred for statistical inference)

Both operators take two input expressions and return a numeric covariance value.

## Basic Covariance Calculation

Calculate covariance between marketing spend and revenue:

```javascript
db.monthlyMetrics.aggregate([
  {
    $setWindowFields: {
      sortBy: { month: 1 },
      output: {
        spendRevenueCovPop: {
          $covariancePop: ["$marketingSpend", "$revenue"],
          window: { documents: ["unbounded", "current"] }
        },
        spendRevenueCovSamp: {
          $covarianceSamp: ["$marketingSpend", "$revenue"],
          window: { documents: ["unbounded", "current"] }
        }
      }
    }
  }
])
```

A large positive covariance suggests that higher marketing spend correlates with higher revenue.

## Covariance Over a Rolling Window

Compute covariance over the last 6 months only:

```javascript
db.monthly.aggregate([
  {
    $setWindowFields: {
      sortBy: { month: 1 },
      output: {
        rollingCov6Month: {
          $covarianceSamp: ["$temperature", "$energyUsage"],
          window: { documents: [-5, 0] }
        }
      }
    }
  }
])
```

This computes the 6-month rolling covariance between temperature and energy usage, useful for detecting seasonal patterns.

## Covariance Across the Full Partition

Omit the `window` key to compute covariance across the entire partition:

```javascript
db.experiments.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$experimentId",
      sortBy: { runId: 1 },
      output: {
        experimentCov: {
          $covariancePop: ["$inputValue", "$outputValue"]
          // no window key - uses entire partition
        }
      }
    }
  }
])
```

Every document in the same experiment gets the same covariance value - the covariance for the whole experiment.

## Normalizing to Correlation Coefficient

Covariance alone can be hard to interpret because it depends on the scale of the variables. Divide by the product of standard deviations to get the Pearson correlation coefficient (-1 to 1):

```javascript
db.data.aggregate([
  {
    $setWindowFields: {
      sortBy: { period: 1 },
      output: {
        covariance: {
          $covarianceSamp: ["$x", "$y"],
          window: { documents: ["unbounded", "current"] }
        },
        stdDevX: {
          $stdDevSamp: "$x",
          window: { documents: ["unbounded", "current"] }
        },
        stdDevY: {
          $stdDevSamp: "$y",
          window: { documents: ["unbounded", "current"] }
        }
      }
    }
  },
  {
    $addFields: {
      correlation: {
        $cond: {
          if: {
            $gt: [{ $multiply: ["$stdDevX", "$stdDevY"] }, 0]
          },
          then: {
            $divide: [
              "$covariance",
              { $multiply: ["$stdDevX", "$stdDevY"] }
            ]
          },
          else: null
        }
      }
    }
  }
])
```

## Partitioned Covariance

Compute covariance independently for each product category:

```javascript
db.products.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$category",
      sortBy: { week: 1 },
      output: {
        priceUnitsCovSamp: {
          $covarianceSamp: ["$price", "$unitsSold"],
          window: { documents: ["unbounded", "current"] }
        }
      }
    }
  }
])
```

## Interpreting Covariance Values

```text
Positive covariance: variables tend to increase/decrease together
Negative covariance: one variable increases as the other decreases
Near-zero covariance: variables are largely independent
```

## Summary

`$covariancePop` and `$covarianceSamp` inside `$setWindowFields` bring statistical covariance calculations to MongoDB aggregation pipelines. Use `$covarianceSamp` for sample-based analysis and `$covariancePop` when analyzing an entire population. Combine covariance with `$stdDevSamp` to compute correlation coefficients for normalized relationship analysis. These operators are particularly valuable for financial modeling, A/B test analysis, and detecting relationships between business metrics.
