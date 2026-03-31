# How to Use multiIf() for Multiple Conditions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Conditional Function, Classification, Data Transformation

Description: Learn how to use multiIf() in ClickHouse to handle multiple conditions cleanly, replacing nested if() or verbose CASE WHEN statements.

---

`multiIf(cond1, result1, cond2, result2, ..., else_result)` evaluates conditions in order and returns the result of the first matching condition. It is equivalent to a chain of nested `if()` calls or a `CASE WHEN` expression. For more than two conditions, `multiIf()` is more readable than nesting `if()` calls and slightly more concise than `CASE WHEN`.

## Basic Syntax

```text
multiIf(cond1, result1, cond2, result2, ..., condN, resultN, else_result)
```

Conditions are evaluated left to right. The result for the first true condition is returned. If no condition matches, `else_result` is returned.

## Basic Usage

```sql
-- Categorize orders into tiers
SELECT
    order_id,
    amount,
    multiIf(
        amount >= 1000, 'enterprise',
        amount >= 500,  'premium',
        amount >= 100,  'standard',
        'basic'
    ) AS order_tier
FROM orders
LIMIT 10;
```

## Replacing Nested if() Calls

Nested `if()` quickly becomes hard to read. `multiIf()` is the cleaner alternative.

```sql
-- Hard to read (nested if):
SELECT
    user_id,
    if(score >= 90, 'A',
       if(score >= 80, 'B',
          if(score >= 70, 'C',
             if(score >= 60, 'D', 'F')))) AS grade
FROM exam_results;

-- Much cleaner with multiIf:
SELECT
    user_id,
    multiIf(
        score >= 90, 'A',
        score >= 80, 'B',
        score >= 70, 'C',
        score >= 60, 'D',
        'F'
    ) AS grade
FROM exam_results
LIMIT 10;
```

## Applying Tiered Discounts

```sql
-- Apply discount rate based on customer segment and order size
SELECT
    order_id,
    customer_segment,
    amount,
    multiIf(
        customer_segment = 'vip'      AND amount >= 500, 0.20,
        customer_segment = 'vip'      AND amount < 500,  0.10,
        customer_segment = 'regular'  AND amount >= 500, 0.05,
        0.0
    ) AS discount_rate,
    amount * (1 - multiIf(
        customer_segment = 'vip'      AND amount >= 500, 0.20,
        customer_segment = 'vip'      AND amount < 500,  0.10,
        customer_segment = 'regular'  AND amount >= 500, 0.05,
        0.0
    )) AS final_price
FROM orders
LIMIT 10;
```

## Complex Classification Logic

```sql
-- Classify HTTP status codes
SELECT
    request_id,
    status_code,
    multiIf(
        status_code >= 500, 'server_error',
        status_code >= 400, 'client_error',
        status_code >= 300, 'redirect',
        status_code >= 200, 'success',
        'informational'
    ) AS status_class
FROM http_logs
LIMIT 20;
```

## Conditional Value Mapping

```sql
-- Map country codes to regions
SELECT
    user_id,
    country_code,
    multiIf(
        country_code IN ('US', 'CA', 'MX'), 'north_america',
        country_code IN ('GB', 'DE', 'FR', 'NL'), 'europe',
        country_code IN ('JP', 'CN', 'KR', 'IN'), 'asia_pacific',
        'other'
    ) AS region
FROM users
LIMIT 10;
```

## Using multiIf() Inside Aggregations

You can use `multiIf()` inside aggregate functions to compute conditional sums or counts.

```sql
-- Break down revenue by tier in a single query
SELECT
    toDate(order_time) AS order_date,
    sum(multiIf(amount >= 1000, amount, 0)) AS enterprise_revenue,
    sum(multiIf(amount >= 500 AND amount < 1000, amount, 0)) AS premium_revenue,
    sum(multiIf(amount < 500, amount, 0)) AS standard_revenue
FROM orders
GROUP BY order_date
ORDER BY order_date DESC
LIMIT 30;
```

## NULL Handling in multiIf

`multiIf()` handles NULL conditions - a NULL condition is treated as false, moving on to the next condition.

```sql
-- Handle potentially NULL values in conditions
SELECT
    user_id,
    multiIf(
        isNull(subscription_tier), 'no subscription',
        subscription_tier = 'gold',   'gold member',
        subscription_tier = 'silver', 'silver member',
        'basic member'
    ) AS membership_label
FROM users
LIMIT 10;
```

## Numeric Classification

```sql
-- Classify response times into performance buckets
SELECT
    request_id,
    response_ms,
    multiIf(
        response_ms < 50,   'fast',
        response_ms < 200,  'acceptable',
        response_ms < 1000, 'slow',
        'very_slow'
    ) AS performance_class
FROM api_requests
LIMIT 20;
```

## Summary

`multiIf()` is the idiomatic ClickHouse way to handle multiple conditions. It is more readable than nested `if()` chains and slightly more concise than `CASE WHEN`. Conditions are evaluated left to right and short-circuit on the first match. Use it for tiered classifications, status code mapping, discount logic, and any scenario requiring more than two conditional branches.
