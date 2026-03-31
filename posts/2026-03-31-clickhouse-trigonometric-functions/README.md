# How to Use sin(), cos(), tan() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Math Function, Trigonometry, Geospatial

Description: Learn how sin(), cos(), and tan() work in ClickHouse with radians-based input, including examples for angle computations, geospatial calculations, and wave generation.

---

ClickHouse provides the standard trigonometric functions `sin()`, `cos()`, and `tan()`, all operating on angles expressed in radians. These functions appear in geospatial distance calculations, signal generation, coordinate transformations, and any analytical work involving cyclic or angular data. Understanding how to convert between degrees and radians is essential when working with geographic coordinates or user-provided degree values.

## Function Signatures

```text
sin(x)   -- sine of x (in radians)
cos(x)   -- cosine of x (in radians)
tan(x)   -- tangent of x (in radians)
```

All three return Float64. Input must be in radians. To convert degrees to radians, multiply by `pi() / 180`.

## Degrees to Radians Conversion

Before using trig functions with degree values, convert using the standard formula.

```text
radians = degrees * pi() / 180
degrees = radians * 180 / pi()
```

```sql
SELECT
    round(sin(pi() / 6), 4)          AS sin_30_deg,
    round(cos(pi() / 3), 4)          AS cos_60_deg,
    round(tan(pi() / 4), 4)          AS tan_45_deg,
    round(sin(45 * pi() / 180), 4)   AS sin_45_deg;
```

`sin(pi()/6)` = 0.5, `cos(pi()/3)` = 0.5, `tan(pi()/4)` = 1.0.

## Haversine Distance Formula

The haversine formula computes great-circle distance between two points on a sphere given their latitudes and longitudes. It uses `sin()`, `cos()`, and inverse trig functions together.

```sql
CREATE TABLE city_coords
(
    city      String,
    latitude  Float64,
    longitude Float64
)
ENGINE = MergeTree
ORDER BY city;

INSERT INTO city_coords VALUES
('New York',    40.7128, -74.0060),
('London',      51.5074,  -0.1278),
('Tokyo',       35.6762, 139.6503),
('Sydney',     -33.8688, 151.2093),
('Paris',       48.8566,   2.3522);
```

Compute haversine distance in kilometers between all city pairs.

```sql
SELECT
    a.city AS city_a,
    b.city AS city_b,
    round(
        2 * 6371 * asin(sqrt(
            pow(sin((b.latitude  - a.latitude)  * pi() / 180 / 2), 2) +
            cos(a.latitude * pi() / 180) *
            cos(b.latitude * pi() / 180) *
            pow(sin((b.longitude - a.longitude) * pi() / 180 / 2), 2)
        )),
    0) AS distance_km
FROM city_coords a
CROSS JOIN city_coords b
WHERE a.city < b.city
ORDER BY distance_km;
```

## Generating Synthetic Wave Data

Use `sin()` to generate test data with cyclic patterns, useful for testing time-series algorithms or creating sample data for dashboards.

```sql
SELECT
    t AS time_step,
    round(sin(2 * pi() * t / 20), 4)               AS sine_wave,
    round(cos(2 * pi() * t / 20), 4)               AS cosine_wave,
    round(sin(2 * pi() * t / 20) * 10 + 50, 2)     AS scaled_signal
FROM (
    SELECT arrayJoin(range(40)) AS t
)
ORDER BY t;
```

## Polar to Cartesian Conversion

Convert polar coordinates (r, theta in degrees) to Cartesian (x, y) using `cos()` and `sin()`.

```sql
SELECT
    r,
    theta_deg,
    round(r * cos(theta_deg * pi() / 180), 4) AS x,
    round(r * sin(theta_deg * pi() / 180), 4) AS y
FROM (
    SELECT
        arrayJoin([1.0, 2.0, 5.0])  AS r,
        arrayJoin([0, 90, 180, 270]) AS theta_deg
)
LIMIT 8;
```

## Detecting Seasonal Patterns

Encode the time-of-year as a cyclic feature using `sin()` and `cos()` on the day-of-year. This preserves the cyclic nature (Dec 31 is close to Jan 1) for machine learning models.

```sql
SELECT
    toDate('2024-01-01') + toIntervalDay(day_num - 1)   AS date_val,
    day_num,
    round(sin(2 * pi() * day_num / 365), 4)             AS sin_day,
    round(cos(2 * pi() * day_num / 365), 4)             AS cos_day
FROM (
    SELECT arrayJoin([1, 91, 182, 274, 365]) AS day_num
);
```

## Summary

`sin()`, `cos()`, and `tan()` in ClickHouse operate on radian inputs. Convert degree values with `x * pi() / 180` before passing to these functions. Use them for geospatial distance calculations (haversine formula), polar-to-Cartesian coordinate conversions, cyclic feature encoding for time-based data, and generating synthetic wave data. All three compose naturally with `pow()`, `sqrt()`, `asin()`, `acos()`, and `atan2()` to implement full trigonometric formulas within SQL queries.
