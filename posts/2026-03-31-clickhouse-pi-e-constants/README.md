# How to Use pi() and e() Constants in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Math Function, Constants, Numeric

Description: Learn how pi() and e() provide mathematical constants in ClickHouse, with examples for degree-radian conversion, geometric calculations, and natural growth formulas.

---

ClickHouse provides two fundamental mathematical constants as functions: `pi()` returns pi (approximately 3.14159265358979) and `e()` returns Euler's number (approximately 2.71828182845905). Using these functions rather than hardcoded literals ensures maximum precision in calculations and makes formulas self-documenting. They appear wherever geometry, trigonometry, probability, and exponential modeling intersect with SQL analytics.

## Function Signatures

```text
pi()   -- returns pi = 3.14159265358979...
e()    -- returns e  = 2.71828182845905...
```

Both return Float64 with full double-precision accuracy.

## Basic Values

```sql
SELECT
    pi()                    AS pi_value,
    e()                     AS e_value,
    round(pi(), 10)         AS pi_10dp,
    round(e(),  10)         AS e_10dp;
```

## Degrees and Radians Conversion

The most common use of `pi()` is converting between degrees and radians. Define the conversion inline in your query.

```text
radians = degrees * pi() / 180
degrees = radians * 180 / pi()
```

```sql
SELECT
    deg,
    round(deg * pi() / 180, 6)                AS radians,
    round(deg * pi() / 180 * 180 / pi(), 6)   AS back_to_degrees
FROM (
    SELECT arrayJoin([0, 30, 45, 60, 90, 120, 180, 270, 360]) AS deg
);
```

## Circle and Sphere Geometry

Use `pi()` for standard geometric formulas involving circles and spheres.

```sql
SELECT
    radius,
    round(pi() * pow(radius, 2), 4)                AS circle_area,
    round(2 * pi() * radius, 4)                    AS circumference,
    round(4.0/3.0 * pi() * pow(radius, 3), 4)      AS sphere_volume,
    round(4 * pi() * pow(radius, 2), 4)            AS sphere_surface_area
FROM (
    SELECT arrayJoin([1.0, 2.5, 5.0, 10.0]) AS radius
);
```

## Natural Exponential Growth

Use `e()` in growth and decay formulas, particularly for continuous compounding models.

```sql
CREATE TABLE investment_scenarios
(
    scenario    String,
    principal   Float64,
    rate        Float64,
    years       Float64
)
ENGINE = MergeTree
ORDER BY scenario;

INSERT INTO investment_scenarios VALUES
('conservative', 10000, 0.03, 10),
('moderate',     10000, 0.06, 10),
('aggressive',   10000, 0.10, 10);
```

Use `e()` instead of `exp()` to make the formula explicit.

```sql
SELECT
    scenario,
    principal,
    rate,
    years,
    round(principal * pow(e(), rate * years), 2)   AS continuous_compound,
    round(principal * exp(rate * years), 2)        AS using_exp_check
FROM investment_scenarios;
```

Both expressions are equivalent; `exp(x)` is shorthand for `pow(e(), x)`.

## Normal Distribution PDF

The probability density function of the standard normal distribution uses both `e()` and `pi()`.

```text
f(x) = (1 / sqrt(2*pi)) * e^(-x^2 / 2)
```

```sql
SELECT
    x,
    round(
        (1.0 / sqrt(2 * pi())) * pow(e(), -pow(x, 2) / 2.0),
    6) AS normal_pdf
FROM (
    SELECT arrayJoin([-3.0, -2.0, -1.0, 0.0, 1.0, 2.0, 3.0]) AS x
);
```

## Using pi() in Cyclic Feature Encoding

Encode time-of-day as cyclic features using `pi()` to define the period.

```sql
SELECT
    hour,
    round(sin(2 * pi() * hour / 24), 4) AS sin_hour,
    round(cos(2 * pi() * hour / 24), 4) AS cos_hour
FROM (
    SELECT arrayJoin([0, 6, 12, 18, 23]) AS hour
);
```

## Euler's Number in Information Theory

Use `e()` and `log()` together in entropy-related formulas. The natural logarithm base-e entropy of a uniform distribution over n outcomes is `log(n)`.

```sql
SELECT
    n,
    round(log(n), 4)                       AS max_entropy_nats,
    round(log(n) / log(2), 4)             AS max_entropy_bits,
    round(1.0 / n, 6)                     AS uniform_prob,
    round(-log(1.0 / n), 4)              AS self_information_nats
FROM (
    SELECT arrayJoin([2, 4, 8, 16, 256]) AS n
);
```

## Summary

`pi()` and `e()` provide full-precision mathematical constants in ClickHouse, eliminating the need to hardcode approximations. Use `pi()` for degree-radian conversion, circle geometry, and cyclic feature encoding. Use `e()` to make exponential growth formulas self-documenting, in the normal distribution PDF alongside `pi()`, and in any context where Euler's number has mathematical significance. Both constants return Float64 and compose naturally with `pow()`, `exp()`, `log()`, `sin()`, `cos()`, and other math functions to implement standard mathematical formulas directly in SQL.
