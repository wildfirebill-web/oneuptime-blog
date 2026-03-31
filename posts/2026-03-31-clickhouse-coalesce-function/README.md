# How to Use coalesce() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, NULL Handling, Conditional Function, Data Cleaning, Fallback

Description: Learn how to use coalesce() in ClickHouse to return the first non-NULL value from a list, enabling multi-source fallbacks and default value chains.

---

`coalesce(x1, x2, ..., xN)` returns the first non-NULL argument from its list. If all arguments are NULL, it returns NULL. It is the standard SQL way to express "use this value, or fall back to that one". ClickHouse fully supports `coalesce()` and it is especially useful when combining data from multiple sources where any individual source may be missing a value.

## Basic Usage

```sql
-- Return the first non-NULL value
SELECT coalesce(NULL, NULL, 'third', 'fourth') AS result;
-- Returns: 'third'

-- Use coalesce to provide a default
SELECT
    user_id,
    coalesce(nickname, first_name, 'Anonymous') AS display_name
FROM users
LIMIT 10;
```

## Multi-Source Data Fallback

When joining tables where some sources have more complete data than others, `coalesce` provides clean fallback logic.

```sql
-- Prefer CRM data over raw import data over default
SELECT
    r.user_id,
    coalesce(crm.email, raw.email, 'unknown@example.com') AS best_email,
    coalesce(crm.phone, raw.phone)                         AS best_phone,
    coalesce(crm.company, raw.company, 'Unknown Company')  AS best_company
FROM raw_contacts AS r
LEFT JOIN crm_contacts AS crm ON r.user_id = crm.user_id
LEFT JOIN raw_imports  AS raw ON r.user_id = raw.user_id
LIMIT 10;
```

## Default Value Chains

Use `coalesce` to build a chain of fallbacks from most specific to most general.

```sql
-- Use the most specific available price
SELECT
    product_id,
    customer_id,
    coalesce(
        customer_specific_price,
        segment_price,
        promotional_price,
        base_price
    ) AS effective_price
FROM price_lookup
LIMIT 10;
```

## Filling Missing Data in Analytics

When analytical data has gaps (missing rows for some time periods), use `coalesce` in combination with a LEFT JOIN to fill them.

```sql
-- Fill missing daily revenue with 0
SELECT
    d.date,
    coalesce(r.revenue, 0) AS daily_revenue
FROM (
    SELECT toDate(number + toDate('2025-01-01')) AS date
    FROM numbers(365)
) AS d
LEFT JOIN (
    SELECT
        toDate(sale_date) AS date,
        sum(amount)       AS revenue
    FROM sales
    WHERE sale_date >= '2025-01-01'
    GROUP BY date
) AS r ON d.date = r.date
ORDER BY d.date;
```

## coalesce vs ifNull

For exactly two arguments, `ifNull` and `coalesce` are equivalent. For three or more, `coalesce` is the right choice.

```sql
-- These are equivalent for two arguments:
SELECT
    coalesce(email, 'no email')     AS a,
    ifNull(email, 'no email')       AS b
FROM users
LIMIT 5;

-- coalesce handles three+ arguments; ifNull cannot:
SELECT
    coalesce(mobile_phone, home_phone, work_phone, 'no phone') AS best_phone
FROM user_contacts
LIMIT 5;
```

## Coalesce in Aggregations

Use `coalesce` inside aggregate functions to ensure consistent results even when some rows have NULL values.

```sql
-- Sum with NULL-safe default
SELECT
    product_id,
    sum(coalesce(discount, 0)) AS total_discount,
    avg(coalesce(rating, 3.0)) AS avg_rating
FROM product_reviews
GROUP BY product_id
ORDER BY product_id
LIMIT 10;
```

## Handling Multi-Table Joins with coalesce

When doing multi-way JOINs where any table might not have a matching row, `coalesce` picks the first available value.

```sql
-- Get the best available address across multiple address tables
SELECT
    c.customer_id,
    coalesce(a1.city, a2.city, a3.city, 'Unknown')     AS city,
    coalesce(a1.country, a2.country, a3.country, 'US') AS country
FROM customers AS c
LEFT JOIN primary_addresses   AS a1 ON c.customer_id = a1.customer_id
LEFT JOIN secondary_addresses AS a2 ON c.customer_id = a2.customer_id
LEFT JOIN billing_addresses   AS a3 ON c.customer_id = a3.customer_id
LIMIT 10;
```

## coalesce with Type Consistency

All arguments to `coalesce` must be of compatible types. ClickHouse will implicitly cast compatible types.

```sql
-- Mixing Nullable and non-nullable is fine
SELECT
    coalesce(toNullable(value_a), value_b) AS result
FROM mixed_sources
LIMIT 5;
```

## Summary

`coalesce()` returns the first non-NULL value from its argument list. It is the standard tool for multi-source fallbacks, default value chains, and filling gaps in data. For exactly two arguments, `ifNull()` is a more concise alternative. For three or more fallback options, `coalesce()` is the right choice. All arguments must have compatible types, and ClickHouse handles implicit type coercion for compatible numeric and string types.
