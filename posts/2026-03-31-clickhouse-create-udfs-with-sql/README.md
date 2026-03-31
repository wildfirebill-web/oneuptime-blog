# How to Create UDFs in ClickHouse with SQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, UDF, SQL, User-Defined Function, Analytics

Description: Learn how to create SQL-based user-defined functions in ClickHouse to encapsulate reusable business logic directly in the database.

---

## SQL UDFs in ClickHouse

ClickHouse supports SQL user-defined functions (UDFs) that let you encapsulate reusable expressions as named functions. SQL UDFs are evaluated inline - they are effectively macros that expand at query time.

## Create a Simple SQL UDF

```sql
CREATE FUNCTION calculate_discount AS (price, discount_pct) ->
    price * (1 - discount_pct / 100);
```

Use it in a query:

```sql
SELECT
    product_id,
    price,
    discount_percent,
    calculate_discount(price, discount_percent) AS discounted_price
FROM products;
```

## UDF for String Manipulation

```sql
CREATE FUNCTION format_full_name AS (first, last) ->
    concat(upper(substring(first, 1, 1)), lower(substring(first, 2)),
           ' ',
           upper(substring(last, 1, 1)), lower(substring(last, 2)));
```

```sql
SELECT format_full_name('john', 'DOE') AS name;
-- Returns: John Doe
```

## UDF for Date Logic

```sql
CREATE FUNCTION is_weekend AS (dt) ->
    toDayOfWeek(dt) IN (6, 7);

SELECT
    event_time,
    is_weekend(event_time) AS on_weekend
FROM events
WHERE event_time >= today() - 7;
```

## UDF for Classification

```sql
CREATE FUNCTION classify_latency AS (ms) ->
    multiIf(
        ms < 100, 'fast',
        ms < 500, 'acceptable',
        ms < 2000, 'slow',
        'very_slow'
    );
```

```sql
SELECT
    request_id,
    latency_ms,
    classify_latency(latency_ms) AS latency_class
FROM api_requests
WHERE request_time >= now() - INTERVAL 1 HOUR;
```

## UDF for Business Metrics

```sql
CREATE FUNCTION calculate_cac AS (marketing_spend, new_customers) ->
    if(new_customers = 0, 0, marketing_spend / new_customers);

CREATE FUNCTION calculate_ltv AS (avg_order_value, purchase_frequency, customer_lifespan_years) ->
    avg_order_value * purchase_frequency * customer_lifespan_years;
```

## List All UDFs

```sql
SELECT
    name,
    create_query
FROM system.functions
WHERE origin = 'SQLUserDefined';
```

## Drop a UDF

```sql
DROP FUNCTION calculate_discount;
```

## Limitations of SQL UDFs

- SQL UDFs cannot contain state or access tables
- They are pure expressions - no loops, conditionals beyond `multiIf`, or subqueries
- For complex logic, use executable UDFs with Python or shell scripts instead

## Replace an Existing UDF

```sql
DROP FUNCTION IF EXISTS classify_latency;

CREATE FUNCTION classify_latency AS (ms) ->
    multiIf(
        ms < 50, 'excellent',
        ms < 200, 'good',
        ms < 1000, 'slow',
        'critical'
    );
```

## Nesting UDFs

SQL UDFs can call other UDFs:

```sql
CREATE FUNCTION hourly_rate AS (daily_rate) -> daily_rate / 24;
CREATE FUNCTION annual_cost AS (daily_rate, days) -> daily_rate * days;
CREATE FUNCTION hourly_cost AS (daily_rate, hours) -> hourly_rate(daily_rate) * hours;
```

## Summary

SQL UDFs in ClickHouse provide a clean way to encapsulate reusable expressions as named functions. They improve query readability, enforce consistent business logic, and can be nested to build more complex transformations. For logic that requires external calls or complex control flow, use executable UDFs instead.
